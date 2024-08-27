[![Website](https://img.shields.io/badge/sqr--094-lsst.io-brightgreen.svg)](https://sqr-094.lsst.io)
[![CI](https://github.com/lsst-sqre/sqr-094/actions/workflows/ci.yaml/badge.svg)](https://github.com/lsst-sqre/sqr-094/actions/workflows/ci.yaml)

# TAP Query History via UWS

## SQR-094

A requirement for the RSP and in particular the portal is providing users the ability to view their past queries and possibly interact with them, for example by re-running, storing or sharing with others.
This note describes a proposed approach to address this functionality via the use of the “jobs” API  which is part of the UWS standard and available via the CADC TAP service.

**Links:**

- Publication URL: https://sqr-094.lsst.io
- Alternative editions: https://sqr-094.lsst.io/v
- GitHub repository: https://github.com/lsst-sqre/sqr-094
- Build system: https://github.com/lsst-sqre/sqr-094/actions/


## Build this technical note

You can clone this repository and build the technote locally if your system has Python 3.11 or later:

```sh
git clone https://github.com/lsst-sqre/sqr-094
cd sqr-094
make init
make html
```

Repeat the `make html` command to rebuild the technote after making changes.
If you need to delete any intermediate files for a clean build, run `make clean`.

The built technote is located at `_build/html/index.html`.

## Publishing changes to the web

This technote is published to https://sqr-094.lsst.io whenever you push changes to the `main` branch on GitHub.
When you push changes to a another branch, a preview of the technote is published to https://sqr-094.lsst.io/v.

## Editing this technical note

The main content of this technote is in `index.md` (a Markdown file parsed as [CommonMark/MyST](https://myst-parser.readthedocs.io/en/latest/index.html)).
Metadata and configuration is in the `technote.toml` file.
For guidance on creating content and information about specifying metadata and configuration, see the Documenteer documentation: https://documenteer.lsst.io/technotes.
