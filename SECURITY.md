# Security Policy — Charmed Airflow Documentation

## Reporting a Vulnerability

The easiest way to report a security issue is through a [GitHub Private Security Report](https://github.com/canonical/charmed-airflow-documentation/security/advisories/new) with a description of the issue, the steps you took to create the issue, affected versions, and, if known, mitigations for the issue.

Alternatively, to report a security issue via email, please email [security@ubuntu.com](mailto:security@ubuntu.com) with a description of the issue, the steps you took to create the issue, affected versions, and, if known, mitigations for the issue.

The [Ubuntu Security disclosure and embargo policy](https://ubuntu.com/security/disclosure-policy) contains more information about what you can expect when you contact us and what we expect from you.

## Supported Versions

This repository contains documentation for Charmed Airflow. It follows the same support lifecycle as the underlying Charmed Airflow product. The product currently ships interim releases; no LTS commitment is made at this time.

| Docs Version | Charmed Airflow Track | Ubuntu Base              | Status          | End of Standard Support |
| ------------ | --------------------- | ------------------------ | --------------- | ----------------------- |
| 3.1          | 3.1/edge              | Ubuntu 24.04 LTS (Noble) | **Pre-release** | TBD                     |

Documentation for older tracks receives no further updates. Users are encouraged to refer to documentation for a supported track.

## Product Lifetime and Support Phases

| Phase                    | Description                                                                  |
| ------------------------ | ---------------------------------------------------------------------------- |
| **Standard Support**     | Active documentation updates, including new features and security guidance.  |
| **Security Maintenance** | Security-related documentation updates only.                                 |
| **End of Life (EOL)**    | No further updates. Users must refer to documentation for a supported track. |

Support periods are defined in the Workflows team support policy. The current `3.1/edge` track is in **Pre-release** and will transition to Standard Support upon stable promotion, followed by Security Maintenance prior to its End of Standard Support date.

## Vulnerability Response

Security vulnerabilities affecting Charmed Airflow are triaged and addressed according to the following severity thresholds, based on NVD CVSS scoring:

| Severity     | CVSS Score | Initial Response | Target Remediation |
| ------------ | ---------- | ---------------- | ------------------ |
| **Critical** | 9.0 – 10.0 | Within 24 hours  | Within 7 days      |
| **High**     | 7.0 – 8.9  | Within 72 hours  | Within 30 days     |
| **Medium**   | 4.0 – 6.9  | Within 2 weeks   | Within 90 days     |
| **Low**      | 0.1 – 3.9  | Best effort      | Best effort        |

All **Critical** and **High** severity vulnerabilities will be remediated or have an active remediation plan in place. Any vulnerability listed in the [CISA Known Exploited Vulnerabilities (KEV) catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog) is treated as highest priority regardless of CVSS score.

Vulnerability findings and advisories are tracked via [GitHub Security Advisories](https://github.com/canonical/charmed-airflow-documentation/security/advisories) and coordinated internally with Canonical's Product Security Incident Response Team (PSIRT).

Note that this repository contains documentation only and does not bundle application code or workloads. Security vulnerabilities in Charmed Airflow itself are addressed in the respective product repositories.
