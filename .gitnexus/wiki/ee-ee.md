# ee — ee

# Enterprise Edition (ee) Module

## Overview

The `ee` module is the organizational root for all **Enterprise Edition** features in Papermark. It serves as a container directory for premium capabilities that are available under commercial license, distinct from the open-source Community Edition.

## Purpose

This module exists to:

- Segregate licensed enterprise functionality from open-source code
- Provide a clear entry point for enterprise-grade features
- Enable modular licensing where certain features require an Enterprise license

## Enterprise Features

The following feature categories are housed within this module:

| Feature | Description |
|---------|-------------|
| **Data Rooms** | Secure document sharing environments with controlled access |
| **Q&A Conversations** | Structured Q&A workflows between document owners and viewers |
| **Advanced Permissions** | Granular role-based access control beyond standard sharing |

## Licensing

> ⚠️ **Important**: This code is copyrighted. Use of this code to host your own instance of app.papermark.com requires a valid [Enterprise license](https://www.papermark.com/enterprise).

Enterprise features are available through:
- [Hosted Enterprise plan](https://www.papermark.com/pricing)
- [Enterprise license](https://www.papermark.com/enterprise) for self-hosted deployments

## Module Structure

```
papermark/
└── ee/                    # Enterprise Edition features
    └── README.md          # This module's documentation
```

## Relationship to Core Application

The `ee` module integrates with the main Papermark application as a feature extension layer. Enterprise features follow the same architectural patterns as core modules but are gated behind license verification.

## Contributing

Enterprise features are developed and maintained by the Papermark team. Contributions to this module are handled through enterprise partnership agreements rather than standard pull requests.