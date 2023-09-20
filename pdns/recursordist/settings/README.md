SETTINGS CODE
=============
This directory contains the code to generate both old-style as new style (YAML) settings code.

Inside this directory, there is a `rust` subdirectory that contains the Rust code and build files.
The Rust code uses CXX for bridging between C++ and Rust.
At the moment of writing, we only call Rust code from C++ and not vice versa.

Additionally, the Rust Serde crate (and specifically Serde-YAML) is used to generatec code to handle YAML.

The entry point for code generation is `generate.py`, which uses `table.py` to produce C++, Rust and .rst files.
See `generate.sh` for some details about the generation process.
This directory also contains a couple of `*-in.*` files which are included into files generated by the generation process.

From the C++ point of view, several namespaces are defined:

* `rust`: This namespace contains the classes defined by CXX.
Note that it is sometimes needed to explicitly name it `::rust`, as the `pdns` namespace also has a subnamespace called `rust`.
* `pdns::rust::settings::rec`: The classes and functions generated by CXX, callable from C++.
* `pdns::settings::rec`: The classes and functions of the settings code implemented in C++. This is mainly the code handling the conversion of old-style definitions to new-style (and vice versa).

Internally, the old-style settings are still used by the recursor code.
Rewriting the existing code to start using the new style settings directly would have made the PR introducing the new-style settings even bigger.
Therefore, we chose to keep the code *using* settings the same.
If a new-style settings YAML file is encountered, it will be parsed, validated and the resulting new-style settings will be used to set the old-style settings to the values found in the YAML file.
In the future, when old-style settings do not need to be supported any longer, the recursor itself can start to use the new style settings directly.

A `rec_control show-yaml [file]` command has been added to show the conversion of old-style settings to the new-style YAML.

This directory
--------------
* `cxxsettings-generated.cc`: generated code that implements the C++ part of the settings code.
* `cxxsettings-private.hh`: private interface used by C++ settings code internally.
* `cxxsettings.hh`: public interface of C++ code to be used by pdns_recursor and rec_control.
* `cxxsupport.cc`: hand written C++ code.
* `docs-new-preamble-in.rst`: non-generated part of new settings docs.
* `docs-old-preamble-in.rst`: non-generated part of old settings docs.
* `generate.py`: the Python script to produce C++, Rust and .rst files based on `table.py`.
* `rust`: the directory containing rust code.
* `rust-bridge-in.rs`: file included in the generated Rust code, placed inside the CXX bridge module.
* `rust-preamble-in.rs`: file included in the generated Rust code as a preamble.
* `table.py`: the definitions of all settings.

`rust` subdirectory
-------------------
* `Cargo.toml`: The definition of the Rust `settings` crate, including its dependencies.
* `build.rs`: `The custom build file used by CXX, see CXX docs.
* `cxx.h`: The generic types used by CXX generated code.
* `lib.rs.h`:  The project specific C++ types generated by CXX.
* `libsettings.a`: The actual static library procuced by this crate.
* `src`: The actual rust code, `lib.rs` is generated, `bridge.rs` and `helpers.rs` are maintained manually.
* `target`: The `cargo` maintained Rust build directory.

The YAML settings are stored in a struct called (on the C++ side) `pdns::rust::settings::rec::Recursorsettings`.
This struct has a substruct for each section defined.
Each section has multiple typed variables.
Below we will tour some parts of the (generated) code.
An example settings file in YAML format:

```yaml
dnssec:
  log_bogus: true
incoming:
  listen:
  - 0.0.0.0:5301
  - '[::]:5301'
logging:
  common_errors: true
  disable_syslog: true
  loglevel: 6
recursor:
  daemon: false
  extended_resolution_errors: true
  socket_dir: /tmp/rec
  threads: 4
webservice:
  address: 127.0.0.1
  allow_from:
  - 0.0.0.0/0
  api_key: secret
  port: 8083
  webserver: true
```

The generated code
------------------
C++, Rust and docmentation generating is done by the `generate.py` Python script using `table.py` as input.
After that, the C++ to Rust bridge code is generated by CXX.
Lets take a look at the `log_bogus` setting.
The source of its definition is in `table.py`:

```python
    {
        'name' : 'log_bogus',
        'section' : 'dnssec',
        'oldname' : 'dnssec-log-bogus',
        'type' : LType.Bool,
        'default' : 'false',
        'help' : 'Log DNSSEC bogus validations',
        'doc' : '''
Log every DNSSEC validation failure.
**Note**: This is not logged per-query but every time records are validated as Bogus.
 ''',
    },
```

The old-style documention generated for this can be found in `../docs/settings.rst`:

```
.. _setting-dnssec-log-bogus:

``dnssec-log-bogus``
~~~~~~~~~~~~~~~~~~~~

-  Boolean
-  Default: no

- YAML setting: :ref:`setting-yaml-dnssec.log_bogus`

Log every DNSSEC validation failure.
**Note**: This is not logged per-query but every time records are validated as Bogus.
```

The new-style documention generated for this can be found in `../docs/yamlsettings.rst`, its name includes the section and it lists a YAML default:

```
.. _setting-yaml-dnssec.log_bogus:

``dnssec.log_bogus``
^^^^^^^^^^^^^^^^^^^^

-  Boolean
-  Default: ``false``

- Old style setting: :ref:`setting-dnssec-log-bogus`

Log every DNSSEC validation failure.
**Note**: This is not logged per-query but every time records are validated as Bogus.
```

The C++ code generated from this entry can be found in `cxxsettings-generated.cc`.
The code to define the old-style settings, which is called by the recursor very early after startup and is a replacement for the hand-written code that defines all settings in the old recursor code.
Using generated code and generated docs makes sure the docs and the actual implementation are consistent.
Something which was not true for the old code in all cases.

```cpp
void pdns::settings::rec::defineOldStyleSettings()
  ...
  ::arg().setSwitch("dnssec-log-bogus", "Log DNSSEC bogus validations") = "no";
  ...
```

There is also code generated to assign the current value of old-style `log_bogus` value to the right field in a `Recursorsettings` struct.
This code is used to convert the currently active settings to a YAML file (`pdns_recursor --config=diff`):
```cpp
void pdns::settings::rec::oldStyleSettingsToBridgeStruct(Recursorsettings& settings)
  ...
  settings.dnssec.log_bogus = arg().mustDo("dnssec-log-bogus");
  ...
```

Plus code to set the old-style setting, given a new-style struct:

```cpp
void pdns::settings::rec::bridgeStructToOldStyleSettings(const Recursorsettings& settings)
  ...
  ::arg().set("dnssec-log-bogus") = to_arg(settings.dnssec.log_bogus);
  ...
```

Lastly, there is code to support converting a old-style settings to a new-style struct found in `pdns::settings::rec::oldKVToBridgeStruct()`.
This code is used to convert old style settings files to YAML (used by `rec_control show-yaml`) and to generate a yaml file with all the defaults values (used by `pdns_recursor --config=default`).
The functions implementing that are `pdns::settings::rec::oldStyleSettingsFileToYaml` and `std::string pdns::settings::rec::defaultsToYaml()`, found in `cxxsupport.cc`.

The Rust code generated by `generate.py` can be found in `rust/src/lib.rs`.
It contains these snippets that define the `log_bogus` field in the section struct `Dnssec` and the `dnssec` field in the `Recursorsettings` struct:

```rust
pub struct Dnssec {
   ...
   #[serde(default, skip_serializing_if = "crate::is_default")]
   log_bogus: bool,
   ...
}
pub struct Recursorsettings {
   ...
   #[serde(default, skip_serializing_if = "crate::is_default")]
   dnssec: Dnssec,
   ...
}
```
The `rust/src/lib.rs` file also contains the generated code to handle Serde defaults.
More details on this can be found in `generate.py`.

The C++ version of the `Recursorsettings` struct and its substructs can be found in the CXX generated `rust/lib.rs.h` file:

```cpp
struct Recursorsettings final {
   ...
   ::pdns::rust::settings::rec::Dnssec dnssec;
   ...
   };
   ...
struct Dnssec final {
  ...
  bool log_bogus;
  ...
};
```

The Rust functions callable from C++ are listed in `rust-bridge-in.rs` which gets included into `rust/src/lib.rs` by `generate.py`
An example is the function

```rust
  fn parse_yaml_string(str: &String) -> Result<Recursorsettings>;
```

Which parses YAML and produces a struct with all the settings.
Settings that are not mentioned in the YAML string wil have their default value.

`rust/lib.rs.h` contains the corresponding C++ prototype, defined in the `pdns::rust::settings::rec` namespace:

```cpp
::pdns::rust::settings::rec::Recursorsettings parse_yaml_string(::rust::String const &str);
```

The C++ function `pdns::settings::rec::readYamlSettings()` defined in `cxxsupport.cc` and called in `../rec-main.cc` calls the Rust function `pdns::rust::settings::rec::parse_yaml_string()` to do the actual YAML parsing.