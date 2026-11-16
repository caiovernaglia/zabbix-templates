# Zabbix Templates

A collection of Zabbix templates for devices and systems not covered by the official Zabbix templates — mainly access control, security and building infrastructure equipment.

The repository layout and template conventions follow the [official Zabbix template guidelines](https://www.zabbix.com/documentation/guidelines/en/template_guidelines).

## Templates

| Template | Category | Zabbix | Version | Collection |
|---|---|---|---|---|
| [Hikvision access control by HTTP](templates/access_control/hikvision_acs_http) | Access control | 8.0 | 8.0-1 | HTTP agent (ISAPI) |

Each template directory contains its own `README.md` with requirements, setup, macros, items and triggers.

## Repository structure

```
zabbix-templates/
├── README.md
├── LICENSE
└── templates/
    └── <category>/
        └── <template_name>/
            ├── template_<category>_<template_name>.yaml
            └── README.md
```

- **`<category>`** groups templates by device or system type (e.g. `access_control`, `net`, `app`).
- **`<template_name>`** is the product followed by the collection method (e.g. `hikvision_acs_http`).
- Template files are exported in **YAML**, the default Zabbix export format.

## Installation

1. Download the `.yaml` file of the template you want from the `templates/` directory.
2. In Zabbix, go to **Data collection → Templates → Import**.
3. Select the file and keep **Create new** and **Update existing** checked.
4. Link the template to your hosts and adjust the macros as described in the template's README.

To update a template, import the new version over the existing one. Templates are matched by UUID, so items, history and linked hosts are preserved.

## Conventions

All templates in this repository follow the same rules:

- **Naming:** `<Product> by <method>` (e.g. *Hikvision access control by HTTP*), without the word "template".
- **Template groups:** `Templates/<Category>` (e.g. `Templates/Access control`).
- **Template tags:** a mandatory `class` tag and at least one `target` tag.
- **Item tags:** `component` tags to group items, plus `resource` where items share a component.
- **Trigger tags:** `scope` (`availability`, `capacity`, `notice`, `security`, …).
- **Credentials:** a dedicated, least-privilege user named `zbx_monitor`. Passwords are stored in **Secret text** macros and are never exported.
- **Thresholds and behaviour** are configurable through user macros, so templates can be tuned per host without editing them.

## Versioning

Each template is versioned independently using the format `<zabbix_version>-<revision>` (e.g. `8.0-1`), stored in the template's **vendor version** field.

Releases are tagged per template:

```
<template_name>/<version>
```

Example: `hikvision_acs_http/8.0-1`.

The version shown in Zabbix (**Data collection → Templates**, *Vendor* and *Version* columns) always matches a tag in this repository.

## Compatibility

Templates are developed and tested on **Zabbix 8.0 LTS**. They may work on earlier versions after adjusting the export header, but this is not tested or supported.

## Contributing

Issues and pull requests are welcome. When reporting a problem, please include:

- Zabbix version
- Template version
- Device model and firmware version
- Relevant item errors from **Monitoring → Latest data**

## Author

**Caio Vernaglia**

## License

See [LICENSE](LICENSE).
