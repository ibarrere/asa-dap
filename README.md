Ansible ASA DAP role 
=========

Disclaimer
------------
*WARNING* Cisco recommends only interacting with DAP via ASDM and that you not update the xml file directly. That's no fun though, and I haven't run into any major issues doing it this way. Be aware, however, that it's a lot easier to break your setup this way, and I highly recommend that you test your rules on a non production firewall before pushing them.

Role Variables
--------------

`defaults/main.yml`:

```
---
xml_src_dir: "/tmp/"
xml_src_file: "dap.xml"
xml_src: "{{ xml_src_dir }}{{ xml_src_file }}"
xml_dest_fs: "disk0:/"
xml_dest_file: "dap.xml"
xml_dest: "{{ xml_dest_fs }}{{ xml_dest_file }}"
python_interpreter: "/usr/bin/python"
dap_default_message: "Please contact your system administrator for assistance."
dap_default_action: "terminate"
```

These are mostly self explanatory and specify where the XML files are generated and pushed to. The default setup generates the dap.xml in /tmp and pushes to disk0:/dap.xml. The `dap_default_message` and `dap_default_action` variables specify the message and action for the DfltAccessPolicy policy.

```
dap_records:
  - name: "EXAMPLE1"
    priority: "100"
    logic_op: "and"
    basic:
      match_policy: match-all
      attr_id: aaa.ldap.memberOf
      attr_value: TEST
      operation: EQ
      type: caseless
    acls:
      - "DAP_PERMIT_EXAMPLE1"
dap_acls:
  - "access-list DAP_PERMIT_EXAMPLE1 extended permit ip object-group EXAMPLE1 object-group EXAMPLE2"
```

The above lists define the behavior of the role. Each variable corresponds to a DAP option available in ASDM. The role loops through the `dap_records` list to construct the dap.xml file. Most of the usual options are available, but I built this role to suit my needs, and if something else is required it should be pretty easy to extend. In the above case, one entry will be created called `EXAMPLE1` with a priority of 100. This entry will match users part of an external LDAP group called `TEST` and will apply the access-list `DAP_PERMIT_EXAMPLE1`. The role loops through the `dap_acls` list and just runs the lines present in it, so any objects/object-groups/etc need to already be configured on the ASA. The `dap_acls` list is not required though, and can be ommitted if you'd rather maintain them by hand.

The role can accept an externally defined dict called `asa_dap_lua` which can specify lua code blocks as defined under the "Advanced" logic section in ASDM. The `asa_dap_lua` dict should consist of a key named after the DAP record name and a value with the desired lua code to apply to that record. If the dict is defined this role will loop through and fill in the lua code for the matching DAP record name. The `basic` dict under the `dap_records` list above can coexist with lua, but be careful as it's easy to create records that over- or under-match that way. Using lua alone, the above example would be accomplished like this:

```
dap_records:
  - name: "EXAMPLE1"
    priority: "100"
    logic_op: "and"
    acls:
      - "DAP_PERMIT_EXAMPLE1"
asa_dap_lua:
  EXAMPLE1: |
    assert(function()
      if (aaa.ldap.memberOf == "TEST") then
          return true
      end
      return false
    end)()
dap_acls:
  - "access-list DAP_PERMIT_EXAMPLE1 extended permit ip object-group EXAMPLE1 object-group EXAMPLE2"
```

If your lua is complex you could create the `asa_dap_lua` dict with a template elsewhere. Each `dap_records` list item can be supplemented with additional dicts to define various lua conditions to keep your logic in the same place.  

Example playbook
----------------

```
- hosts: all 
  connection: network_cli

  roles:
    - asa-dap-lua
    - asa-dap
```

Where asa-dap-lua is another role that generates the `asa_dap_lua` dict. 

You can specify the tag `preserve_dap` if you do not want to overwrite the existing "dynamic-access-policy-record" lines on the ASA. Otherwise, the role will run the command `clear conf dynamic-access-policy-record` by default.

Dependencies
------------

This has been tested with ASA version 9.12(2)9

License
-------

BSD
