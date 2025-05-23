---
mapped_pages:
  - https://www.elastic.co/guide/en/security/current/endpoint-self-protection.html
  - https://www.elastic.co/guide/en/serverless/current/security-endpoint-self-protection.html
applies_to:
  stack: all
  serverless:
    security: all
products:
  - id: security
  - id: cloud-serverless
---

# {{elastic-endpoint}} self-protection features [endpoint-self-protection]

{{elastic-endpoint}}, the installed component that performs {{elastic-defend}}'s threat monitoring and prevention, protects itself against users and attackers that may try to interfere with its functionality. Protection features are consistently enhanced to prevent attackers who may attempt to use newer, more sophisticated tactics to interfere with the {{elastic-endpoint}}. Self-protection is enabled by default when {{elastic-endpoint}} installs on supported platforms, listed below.

Self-protection is enabled on the following 64-bit Windows versions:

* Windows 8.1
* Windows 10
* Windows 11
* Windows Server 2012 R2
* Windows Server 2016
* Windows Server 2019
* Windows Server 2022

Self-protection is also enabled on the following macOS versions:

* macOS 10.15 (Catalina)
* macOS 11 (Big Sur)
* macOS 12 (Monterey)

::::{note}
Other Windows and macOS variants (and all Linux distributions) do not have self-protection.
::::


For {{stack}} version >= 7.11.0, self-protection defines the following permissions:

* Users — even Administrator/root — **cannot** delete {{elastic-endpoint}} files (located at `c:\Program Files\Elastic\Endpoint` on Windows, and `/Library/Elastic/Endpoint` on macOS).
* Users **cannot** terminate the {{elastic-endpoint}} program or service.
* Administrator/root users **can** read {{elastic-endpoint}}'s files. On Windows, the easiest way to read {{elastic-endpoint}} files is to start an Administrator `cmd.exe` prompt. On macOS, an Administrator can use the `sudo` command.
* Administrator/root users **can** stop the {{elastic-agent}}'s service. On Windows, run the `sc stop "Elastic Agent"` command. On macOS, run the `sudo launchctl stop elastic-agent` command.
