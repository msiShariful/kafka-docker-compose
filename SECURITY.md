# Security Policy

## Supported Versions

This project is actively maintained on the `master` branch. Security fixes are applied to the latest version only.

| Version | Supported |
| ------- | --------- |
| latest (`master`) | :white_check_mark: |
| older commits | :x: |

## Reporting a Vulnerability

If you discover a security vulnerability, please report it privately:

- Use GitHub's [private vulnerability reporting](https://docs.github.com/en/code-security/security-advisories/guidance-on-reporting-and-writing-information-about-vulnerabilities/privately-reporting-a-security-vulnerability) ("Report a vulnerability" under the **Security** tab), **or**
- Email the maintainer at **nipon@6amtech.com**.

Please include steps to reproduce, affected versions, and potential impact. You can expect an initial response within **5 business days**. Please do not open a public issue for security reports.

## Security Considerations

- All listeners use `PLAINTEXT` with no authentication, authorization, or TLS. This configuration is intended for **local/development use only**.
- Do not expose the broker host ports (`29092`, `39092`, `49092`) or the Kafka-UI port (`9021`) to untrusted networks.
- For any production or networked deployment, enable SASL authentication and TLS encryption, configure ACLs, secure or rotate the `CLUSTER_ID`, and place Kafka-UI behind authentication.
