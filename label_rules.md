# Labeling rules

The annotated tool (`tool_ground_truth`) for every query in datasets A and B follows the
rules below. Labels come from these rules applied to template families written with the help
of Claude Code. No language model labels individual queries, and the labels have not been
verified by hand.

## The 11 tools

| Role | Tools |
|---|---|
| attacker | SSHConnect, NmapScan, PortScan, CheckVulnerability |
| defender | ReadAuthLog, ListeningPorts, BlockIP, CheckFailedLogins, ListProcesses |
| shared | GetSystemInfo, ReadSyslog |

## Rules

Each query has one label. When a query could fit two tools, the tie-break below decides.

| If the query asks for | Label |
|---|---|
| open ports only, or "what is open" | PortScan |
| services, software, or versions on a target | NmapScan |
| whether a service or version is exploitable, or its CVEs | CheckVulnerability |
| opening a remote shell or connection to a host | SSHConnect |
| failed or bad login attempts, password guessing | CheckFailedLogins |
| the authentication log as a whole | ReadAuthLog |
| the general system log | ReadSyslog |
| sockets or ports this host is listening on | ListeningPorts |
| running programs, or what is using CPU | ListProcesses |
| hostname, kernel, OS, or uptime | GetSystemInfo |
| blocking or denying an address | BlockIP |

A request with several steps is labeled by its first (primary) step.

Two more tie-breaks used in dataset B:

- A request that names a remote target (an address, "the target", "the defender") uses the
  attacker tools (for example "what is running on the target?" is NmapScan). A request about
  "this host", "this box" or "this machine" uses the local defender or shared tools (for
  example "what is running on this host?" is ListProcesses).
- A request for "the logs" with no other cue is labeled ReadSyslog.

## Conventions worth knowing

- Dataset A labels "run nmap -p ..." queries as PortScan, because the request is for ports.
  The word "nmap" is a surface cue for NmapScan, so a model can reasonably pick NmapScan.
  Part of the NmapScan vs PortScan disagreement in the data comes from this convention and
  not only from model error. Dataset B keeps the same convention.
- The label is the intended tool under these rules, not the only defensible reading. Queries
  in the `ambiguous` category were written so that more than one tool could plausibly apply.
