# Adding a new OVN feature to the DDlog version of ovn-northd

This document describes the usual steps an OVN developer should go through
when adding a new feature to `ovn-norhtd-ddlog`. In order to make things
less abstract we will use the IP Multicast `ovn-northd-ddlog` implementation
as an example. Even though the document is structured as a tutorial there
might still exist feature specific aspects that are not covered here.


## Overview

DDlog is a dataflow system.  It is triggered by an update to one or more of its
input relations (in `ovn-northd-ddlog` these are linked to NB and SB database
tables).  It then follows the rules in the DDlog program to compute updates to
output relations (also linked to OVSDB tables), which typically involves
updating some intermediate relations (i.e., relations declared without `input`
or `output` qualifiers) in the process:

```
from OVSDB +----------+   +-----------------+   +-----------+ to OVSDB
---------->|Input rels|-->|Intermediate rels|-->|Output rels|---------->
           +----------+   +-----------------+   +-----------+
```

TODO: the above description ignores the auto-generated plumbing between DDlog
and OVSDB tables.  Document this plumbing once it has stabilized.

Consequently, adding a new feature in `ovn-northd` usually involves
the following steps:

1. Update NB and/or SB OVSDB schemas.
1. Configure DDlog/OVSDB bindings.
1. Define intermediate DDlog relations and rules to compute them.
1. Write rules to update output relations.
1. Generate `Logical_Flow`s and/or other forwarding records
   (e.g., `Multicast_Group`) that will control the dataplane operations.

## Update NB and/or SB OVSDB schemas

This step is no different from the normal development flow in C.

Most of the times a developer chooses between two ways of configuring a new
feature:
1. Adding a set of columns to tables in the NB and/or SB database (or
   adding key-value pairs to existing columns).
1. Adding new tables to the NB and/or SB database.

Looking at IP Multicast, there are two `OVN Northbound` tables where
configuration information is stored:
- `Logical_Switch`, column `other_config`, keys `mcast_*`.
- `Logical_Router`, column `options`, keys `mcast_*`.

These tables become inputs to the DDlog pipeline.

In addition a new table, called `IP_Multicast`, is introduced to the SB
database.  This table will be updated by DDlog, i.e., it is an output
of the above pipeline.

## Configuring DDlog/OVSDB bindings

### Configuring `northd/automake.mk`

The `ovsdb2ddlog` compiler ran by `northd/automake.mk` will parse
`ovn-nb.ovsschema` and `ovn-sb.ovsschema` and automatically generate new
columns/relations in `OVN_Northbound.dl` and `OVN_Southbound.dl`.
Additionally DDlog needs to know some attributes of OVSDB tables that
cannot be extracted from the OVSDB schema.  These attributes must
be specified as `ovsdb2ddlog` command-line arguments.

**TODO:** update these instructions to use config file instead of CLI
arguments.

First we need to instruct the `ovsdb2ddlog` compiler to generate output
relations for newly introduced output tables, in this case just the
`IP_Multicast` table.  This is done using the `-o` switch. To optimize
things, we can specify a subset of output table columns to be used as
the table's primary key using `-k` switch.

**TODO:** remove the description of primary key attributes once they are
no longer needed

Finally, in some cases `ovn-northd-ddlog` shouldn't change values
in OVSDB columns that are managed externally (e.g., by a CMS through
`ovn-sbctl`). One such case is the `seq_no` column in the
`IP_Multicast` table. To do that we need to instruct `ovsdb2ddlog` to treat
the column as read-only in `northd/automake.mk` by using the `--ro` switch:

The resulting `IP_Multicast` table configuration in `northd/automake.mk` looks
as follows:

```
northd/OVN_Southbound.dl: ovn-sb.ovsschema
    ovsdb2ddlog -f ovn-sb.ovsschema      \
                [...]                    \
                -o IP_Multicast          \
                -k IP_Multicast.datapath \
                --ro IP_Multicast.seq_no
```

This results in the following autogenerated relation in `OVN_Southbound.dl`,
which needs to be populated by the developer with IP multicast configuration
for each datapath:

```
relation Out_IP_Multicast (
    datapath: string,
    enabled: Set<bool>,
    querier: Set<bool>
)
```

A number of additional tables are generated.  These tables are used by the
auto-generated OVSDB adapter logic and are irrelevant to most DDLog
developers. We list them in [Appendix A]().

### Configuring `ovn-northd-ddlog.c`

DDlog computed deltas for an OVSDB table are not automatically committed
to the `OVN Northbound` or `OVN Southbound` databases unless they are
explicitly added to `get_sb_ops()` in `ovn-northd-ddlog.c`:

```
static struct json *
get_sb_ops(struct northd_ctx *ctx)
{
    struct ds ds = DS_EMPTY_INITIALIZER;

    [...]

    ddlog_table_update(&ds, ctx->ddlog, "OVN_Southbound", "IP_Multicast");

    [...]
}
```

## Define intermediate DDlog relations and rules to compute them.
Obviously there will be a one-to-one relationship between logical
switches/routers and IP multicast configuration. One way to represent this
relationship is to create multicast configuration DDlog relations to be
referenced by `&Switch` and `&Router` DDlog records.

```
/* IP Multicast per switch configuration. */
relation &McastSwitchCfg(
    datapath      : uuid,
    enabled       : bool,
    querier       : bool
}

&McastSwitchCfg(
        .datapath = ls_uuid,
        .enabled  = map_get_bool_def(other_config, "mcast_snoop", false),
        .querier  = map_get_bool_def(other_config, "mcast_querier", true)) :-
    nb.Logical_Switch(._uuid        = ls_uuid,
                      .other_config = other_config).
```

Then reference these relations in `&Switch` and `&Router`. For example, in
`lswitch.dl`, the `&Switch` relation definition now contains:

```
relation &Switch(
    ls:                nb.Logical_Switch,
    [...]
    mcast_cfg:         Ref<McastSwitchCfg>
)
```

And is populated by the following rule which references the correct
`McastSwitchCfg` based on the logical switch uuid:

```
&Switch(.ls        = ls,
        [...]
        .mcast_cfg = mcast_cfg) :-
    nb.Logical_Switch[ls],
    [...]
    mcast_cfg in &McastSwitchCfg(.datapath = ls._uuid).
```

### Build state based on information dynamically updated by `ovn-controller`

Some OVN features rely on information learned by `ovn-controller` to generate
`Logical_Flow`s or other records that control the dataplane. In case of IP
Multicast, `ovn-controller` uses IGMP to learn multicast groups that are joined
by hosts.

Each `ovn-controller` maintains its own set of records to avoid ownership
and concurrency with other controllers. If two hosts that are connected to
the same logical switch but reside on different hypervisors (different
`ovn-controller`s) join the same multicast group `G`, each of the controllers
will create an `IGMP_Group` record in the `OVN Southbound` database which
will contain a set of ports to which the interested hosts are connected.

At this point `ovn-northd-ddlog` will need to aggregate the per-chassis
IGMP records in order to generate a single `Logical_Flow` for the IP multicast
group `G`. Moreover, the ports on which the hosts are connected are
represented as references to `Port_Binding` records in the database. These
also need to be translated to `&SwitchPort` DDlog relations. The corresponding
DDlog operations that need to be performed are:

- Flatten the `<IGMP group, ports>` mapping in order to be able to do the
  translation from `Port_Binding` to `&SwitchPort`. For each `IGMP_Group`
  record in the `OVN Southbound` database generate an individual record
  of type `IgmpSwitchGroupPort` for each `Port_Binding` in the set of ports
  that joined the group. Also, translate the `Port_Binding` uuid to the
  corresponding `Logical_Switch_Port` uuid:
    ```
    relation IgmpSwitchGroupPort(
        address: string,
        switch : Ref<Switch>,
        port   : uuid
    )

    IgmpSwitchGroupPort(address, switch, lsp_uuid) :-
        sb.IGMP_Group(.address = address, .datapath = igmp_dp_set,
                      .ports = pb_ports),
        var pb_port_uuid = FlatMap(pb_ports),
        sb.Port_Binding(._uuid = pb_port_uuid, .logical_port = lsp_name),
        &SwitchPort(
            .lsp = nb.Logical_Switch_Port{._uuid = lsp_uuid, .name = lsp_name},
            .sw = switch).
    ```
- Aggregate the flattened IgmpSwitchGroupPort (implicitly from all
  `ovn-controller` instances) grouping by adress and logical switch:
    ```
    relation IgmpSwitchMulticastGroup(
        address: string,
        switch : Ref<Switch>,
        ports  : Set<uuid>
    )

    IgmpSwitchMulticastGroup(address, switch, ports) :-
        IgmpSwitchGroupPort(address, switch, port),
        var ports = Aggregate((address, switch), group2set(port)).
    ```

At this point we have all the feature configuration relevant information
stored in DDlog relations in `ovn-northd-ddlog` memory.

## Write rules to update output relations

The developer updates output tables by writing rules that
generate `Out_*` relations. For IP Multicast this means:

```
/* IP_Multicast table (only applicable for Switches). */
sb.Out_IP_Multicast(.datapath = uuid2name(cfg.datapath),
                    .enabled = set_singleton(cfg.enabled),
                    .querier = set_singleton(cfg.querier)) :-
    &McastSwitchCfg[cfg].
```

Note: `OVN_Southbound.dl` also contains an `IP_Multicast` relation with
`input` qualifier.  This relation stores the current snapshot of the OVSDB
tabl and cannot be written to.

## Generate `Logical_Flow`s and/or other forwarding records

At this point we have defined all DDlog relations required to generate
`Logical_Flow`s. All we have to do is write the rules to do so. For each
`IgmpSwitchMulticastGroup` we generate a `Flow` that has as action
`"outport = <Multicast_Group>; output;"`:

```
/* Ingress table 17: Add IP multicast flows learnt from IGMP (priority 90). */
for (IgmpSwitchMulticastGroup(.address = address, .switch = &sw)) {
    Flow(.logical_datapath = sw.dpname,
         .stage            = switch_stage(IN, L2_LKUP),
         .priority         = 90,
         .__match          = "eth.mcast && ip4 && ip4.dst == ${address}",
         .actions          = "outport = \"${address}\"; output;",
         .external_ids     = map_empty())
}
```

In some cases generating a logical flow is not enough. For IGMP
we also need to maintain `OVN Southbound` `Multicast_Group` records, one
per IGMP group storing the corresponding `Port_Binding` uuids of ports where
multicast traffic should be sent. This is also relatively straightforward:

```
/* Create a multicast group for each IGMP group learned by a Switch.
 * 'tunnel_key' == 0 triggers an ID allocation later.
 */
sb.Out_Multicast_Group (.datapath   = switch.dpname,
                        .name       = address,
                        .tunnel_key = 0,
                        .ports      = set_map_uuid2name(port_ids)) :-
    IgmpSwitchMulticastGroup(address, &switch, port_ids).
```

At this point the `Out_Multicast_Group` records will store all the
`Multicast_Group`s that need to eventually exist in the database. However,
the `tunnel_key` is still 0 for all the entries. As `tunnel_key`s need to be
unique we need to adjust the DDlog program to allocate them. We need to:

- Instruct `ovsdb2ddlog` to generate a proxy table for `Multicast_Group`. This
  table is used as an intermediate output table where store records once
  their `tunnel_key` is allocated:

**TODO:** explain this in more detail once the feature has stabilized.

    ```
    northd/OVN_Southbound.dl: ovn-sb.ovsschema
        ovsdb2ddlog -f ovn-sb.ovsschema \
        [...]                           \
        -p Multicast_Group
    ```
- Use the proxy relation to allocate `tunnel_key`s:
    ```
    sb.OutProxy_Multicast_Group(.datapath = mcgroup.datapath,
                                .name = mcgroup.name,
                                .tunnel_key = tunnel_key,
                                .ports = mcgroup.ports) :-
            sb.Swizzled_Multicast_Group[mcgroup],
        mcgroup.tunnel_key == 0,
        MulticastGroupTunKeyAllocation(mcgroup.name, tunnel_key).
    ```
- Define DDlog relations that will allocate `tunnel_key`s. There are two
  cases: tunnel keys for records that already existed in the database are
  preserved to implement stable id allocation; new multicast groups need new
  keys:
    ```
    // Perform the allocation
    relation MulticastGroupTunKeyAllocation(group: string, tunkey: integer)

    // transfer existing allocations from the realized table
    MulticastGroupTunKeyAllocation(group, tunkey) :-
        sb.UUIDMap_Datapath_Binding(_, Left{datapath_uuid}),
        sb.Multicast_Group(.name = group,
                        .datapath = datapath_uuid,
                        .tunnel_key = tunkey).

    // AllocatedMulticastGroupTunKeys(datapath) is not empty (i.e.,
    // contains a single record that stores a set of allocated keys)
    MulticastGroupTunKeyAllocation(group, tunkey) :-
        AllocatedMulticastGroupTunKeys(datapath, allocated),
        NotYetAllocatedMulticastGroupTunKeys(datapath, unallocated),
        (_, var min_key) = mC_IP_MCAST_MIN(),
        (_, var max_key) = mC_IP_MCAST_MAX(),
        var allocation = FlatMap(allocate(allocated, unallocated,
                                          min_key, max_key)),
        (var group, var tunkey) = allocation.

    // AllocatedMulticastGroupTunKeys(datapath) relation is empty
    MulticastGroupTunKeyAllocation(group, tunkey) :-
        NotYetAllocatedMulticastGroupTunKeys(datapath, unallocated),
        not AllocatedMulticastGroupTunKeys(datapath, _),
        (_, var min_key) = mC_IP_MCAST_MIN(),
        (_, var max_key) = mC_IP_MCAST_MAX(),
        var allocation = FlatMap(allocate(set_empty(): Set<bit<64>>,
                                          unallocated, min_key, max_key)),
        (var group, var tunkey) = allocation.
    ```
- Define two more DDlog relations to track the already "allocated tunnel keys"
  and "available tunnel keys":
    ```
    // all tunnel keys already in use in the Realized table
    relation AllocatedMulticastGroupTunKeys(datapath: string,
                                            keys: Set<integer>)

    AllocatedMulticastGroupTunKeys(datapath, keys) :-
        sb.Multicast_Group(.datapath = datapath_uuid, .tunnel_key = tunkey),
        sb.UUIDMap_Datapath_Binding(datapath, Left{datapath_uuid}),
        var keys = Aggregate((datapath), group2set(tunkey)).

    // Multicast_Group's not yet in the Realized table
    relation NotYetAllocatedMulticastGroupTunKeys(datapath: string,
                                                  all_logical_ids: Vec<string>)

    NotYetAllocatedMulticastGroupTunKeys(datapath, all_names) :-
        sb.Out_Multicast_Group(.name = name, .datapath = datapath,
                            .tunnel_key = tunnel_key),
        sb.UUIDMap_Datapath_Binding(datapath, Left{datapath_uuid}),
        not sb.Multicast_Group(.name = name, .datapath = datapath_uuid),
        var all_names = Aggregate((datapath), group2vec(name)).
    ```

## Appendix A. Additional relations generated by `ovsdb2ddlog`

- `Swizzled_IP_Multicast` used by the DDlog program to track all the current
  records (based on the input database records) wherever the uuid of the
  entries is known:
    ```
    primary key (x) x._uuid
    relation Swizzled_IP_Multicast (
        datapath: uuid_or_string_t,
        enabled: Set<bool>,
        querier: Set<bool>
    )
    Swizzled_IP_Multicast(__id_datapath, enabled, querier) :-
        Out_IP_Multicast(datapath, enabled, querier),
        UUIDMap_Datapath_Binding(datapath, __id_datapath).
    ```
- `DeltaPlus_IP_Multicast` used by the DDlog program to track new records that
  are not yet added to the database:
    ```
    output relation DeltaPlus_IP_Multicast (
        datapath: uuid_or_string_t,
        enabled: Set<bool>,
        querier: Set<bool>
    )
    DeltaPlus_IP_Multicast(datapath, enabled, querier) :-
        Swizzled_IP_Multicast(datapath, enabled, querier),
        not IP_Multicast(_, extract_uuid(datapath), _, _).
    ```
- `DeltaMinus_IP_Multicast` used by the DDlog program to track records that
  are no longer needed in the database and need to be removed:
    ```
    output relation DeltaMinus_IP_Multicast (
        _uuid: uuid
    )
    DeltaMinus_IP_Multicast(_uuid) :-
        IP_Multicast(_uuid, datapath, _, _),
        not Swizzled_IP_Multicast(Left{datapath}, _, _).
    ```
- `Update_IP_Multicast` used by the DDlog program to track records whose
  fields need to be updated in the database:
   ```
   output relation Update_IP_Multicast (
       _uuid: uuid,
       enabled: Set<bool>,
       querier: Set<bool>
   )
   Update_IP_Multicast(_uuid, __new_enabled, __new_querier) :-
       Swizzled_IP_Multicast(Left{datapath}, __new_enabled, __new_querier),
       IP_Multicast(_uuid, datapath, __old_enabled, __old_querier),
       (__old_enabled, __old_querier) != (__new_enabled, __new_querier).
   ```