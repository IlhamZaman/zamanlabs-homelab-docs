<!--
Organized from: STEP-17-HOGWARTS-SPELLS-ENCHANTMENTS-AND-TERMINOLOGY.txt
Source wording and technical details were preserved as closely as possible.
-->

> [!CAUTION]
> This is a historical implementation record. Verify versions, addresses, paths, and commands against the live environment before applying changes.

# STEP 17 — HOGWARTS SPELLS, ENCHANTMENTS, AND TERMINOLOGY

Compiled: 2026-08-03 00:29:21 EDT
Operator and Headmaster: Ilham Zaman
Keeper: Harry Potter through Hermes Agent

### PURPOSE

This STEP 17 record lists the Hogwarts spells Ilham taught or developed with
Harry, explains what each one does, records the known Hogwarts enchantments,
and preserves the complete Harry Potter technical terminology at the bottom.

### CLASSIFICATION

Spell
  A reusable Hermes Skill with one clear purpose.

Enchantment
  A scheduled task, cron job, systemd timer, or other persistent automation.
  An enchantment may invoke one or more incantations.

Incantation
  The command, script, playbook, API call, or workflow that actually performs
  work. An incantation is not automatically a spell or an enchantment.


## PART I — SPELLS ILHAM TAUGHT OR DEVELOPED

The live Hogwarts spellbook currently contains the following twelve spells.

1. AUROR'S INSTINCT
   Hermes Skill: aurors-instinct

   What it does:
   Applies Harry's permanent protect-first decision gate before recommendations,
   designs, approvals, or actions. It requires consultation of Hogwarts history
   and topology, consideration of wards and blast radius, usable recovery and
   verification, and preference for the smallest reliable approach.

2. THE AUROR'S INVESTIGATION
   Hermes Skill: aurors-investigation

   What it does:
   Provides the strictly read-only investigation method for failed services,
   unexpected behaviour, security concerns, performance problems, incidents,
   and other Dark Artifacts. It moves through observation, understanding,
   theories, elimination, conclusion, and recommendations, then stops for the
   Headmaster instead of applying a fix.

3. THE HEADMASTER'S APPROVAL
   Hermes Skill: headmasters-approval

   What it does:
   Governs every state-changing operation in Hogwarts. It requires an
   evidence-backed decision brief covering scope, options, risks, affected
   systems, recovery, rollback, verification, and stop conditions. Harry must
   then wait until Ilham explicitly approves the exact operation and stage.

4. HOGWARTS ADVENTURE LIFECYCLE
   Hermes Skill: hogwarts-adventure-lifecycle

   What it does:
   Coordinates a significant investigation, change, incident, migration,
   recovery, architecture decision, or completed objective from evidence
   gathering through approval, safe execution, Patronus verification, factual
   record updates, and reflection. It prevents investigation, design,
   implementation, memory, and wisdom from being confused with one another.

5. KEEPER OF HOGWARTS
   Hermes Skill: keeper-of-hogwarts

   What it does:
   Establishes Hogwarts as Ilham's actual homelab and gives Harry the connected,
   history-aware, evidence-first operating discipline needed to understand and
   protect it. It preserves canonical terminology while keeping exact technical
   names, paths, addresses, services, and errors clear.

6. THE MARAUDER'S MAP
   Hermes Skill: hogwarts-marauders-map

   What it does:
   Loads and maintains the connected, evidence-backed graph of Hogwarts:
   physical devices, The Castle Foundations, houses, networks, addresses,
   storage, power, services, DNS, reverse proxies, automation, monitoring,
   wards, corridors, and dependencies. It records verified current state rather
   than filling gaps with assumptions.

7. HOGWARTS NETWORK IDENTITY INVENTORY
   Hermes Skill: hogwarts-network-identity-inventory

   What it does:
   Maintains the verified ledger of hostnames, IPv4 and IPv6 addresses, and MAC
   addresses. It supports identity lookup, recording, and auditing without
   guessing from stale ARP or neighbour-cache evidence and without changing
   network configuration.

8. THE CASTLE WATCH
   Hermes Skill: castle-watch

   What it does:
   During an authorized inspection, quietly compares verified live evidence
   against the Marauder's Map, Pensieve history, and Ministry's Records. It
   detects meaningful change or drift, classifies intent when evidence allows,
   and alerts Ilham only when safety, health, recovery, or historical consistency
   may be at risk. It is not continuous autonomous monitoring.

9. THE AUROR'S JOURNAL
   Hermes Skill: aurors-journal

   What it does:
   Preserves brief append-only reflections after significant completed Hogwarts
   work when genuine wisdom was earned. It records judgment, restraint,
   uncertainty, mistakes, corrections, and durable lessons without replacing
   the Pensieve's factual adventure history or rewriting earlier entries.

10. HOMELAB INFRASTRUCTURE OPERATIONS
    Hermes Skill: homelab-infrastructure-operations

    What it does:
    Provides the practical procedures for safely documenting, inspecting,
    maintaining, and troubleshooting Hogwarts across Proxmox VE, TrueNAS,
    Linux VMs and LXCs, DNS, reverse proxies, monitoring, storage, backups,
    networking, and automation.

11. PROTEGO MAXIMA
    Hermes Skill: protego-maxima

    What it does:
    Performs a permanent read-only safety review of Terraform plan evidence that
    Ilham manually generated or that a trusted observer delivered. It enumerates
    actions, detects destruction, replacement, dangerous modifications, drift,
    incomplete evidence, and Hogwarts blast radius. It never runs Terraform,
    changes Terraform artifacts, applies a plan, or grants approval.

12. SAFE COMMAND OBSERVERS
    Hermes Skill: safe-command-observers

    What it does:
    Provides the safety architecture for observing manually initiated command
    output without replacing the ordinary command path. It requires explicit
    launchers, interactive gates, at-most-once execution, bounded capture,
    local redaction, status preservation, constrained delivery, independent
    baselines, targeted rollback, and honest acknowledgement boundaries. The
    Watching Portrait was designed and repaired using this spell.


## PART II — ENCHANTMENTS

## 1. HOGWARTS WEEKLY MAINTENANCE

   Classification:
   Recurring scheduled enchantment managed by Hermes cron.

   Schedule:
   Every Sunday at 00:00 in America/New_York.
   Cron expression: 0 0 * * 0

   What it does:
   The House-Elf collector verifies that it reached Alma-MGMT as user izaman,
   validates the reviewed metadata and SHA-256 of /usr/bin/maintenance, checks
   that no conflicting maintenance or package operation is running, and then
   invokes this exact incantation without sudo:

```text
     ssh -n -o BatchMode=yes alma-mgmt /usr/bin/maintenance
```

   It saves the complete raw maintenance log privately, calculates its checksum,
   extracts bounded warning, failure, update, reboot, storage, backup, and recap
   findings, and gives only those filtered findings to the reporting agent. The
   agent uses Keeper of Hogwarts and Homelab Infrastructure Operations to produce
   a concise Hogwarts report delivered to the originating Discord conversation.

   Important boundaries:
   - The collector runs /usr/bin/maintenance directly as izaman, never with sudo.
   - It claims one run per scheduled date to prevent duplicate maintenance.
   - It preserves the maintenance exit status in its evidence.
   - Full raw output remains local; the model receives filtered findings.
   - It does not automatically troubleshoot or broaden the maintenance scope.

   Verified scheduler state when STEP 17 was compiled:
   Enabled, recurring forever, last run status OK on 2026-08-02, with the next
   scheduled run on 2026-08-09 at 00:00 EDT.

## 2. THE WATCHING PORTRAIT

   Technical name:
   watching-portrait-plan

   Classification:
   Persistent, manually invoked, confirmation-gated enchantment. It is not a
   cron job and does not run continuously.

   What it does:
   Allows Ilham to deliberately run the separate observer command:

     watching-portrait-plan plan [Terraform plan arguments]

   The launcher validates the authorized interactive context, permits only the
   literal Terraform subcommand plan, rejects reviewed bypass and artifact-writing
   options, displays the exact command safely, and requires a fresh random
   confirmation phrase entered through /dev/tty. After confirmation, the local
   launcher executes /usr/bin/terraform exactly once using the validated argument
   vector.

   It captures stdout and stderr through separate bounded pipes, continues
   draining after the evidence limit, replays output to the original streams,
   records when evidence is truncated, redacts locally, creates a typed and
   hashed evidence envelope, and attempts one constrained delivery. Protego
   Maxima then reviews the redacted Terraform plan evidence and sends its warning
   or finding only to Ilham's verified private Discord destination.

   Important boundaries:
   - Normal terraform still resolves directly to /usr/bin/terraform.
   - The Watching Portrait never aliases or silently intercepts terraform.
   - It accepts only plan; it does not accept apply, destroy, or other subcommands.
   - The receiver, webhook, Hermes route, and Protego Maxima cannot run Terraform.
   - Delivery or analysis failure cannot rerun Terraform.
   - Terraform's saved exit status remains authoritative.
   - Raw evidence is bounded; only locally redacted evidence may be persisted or
     transported.
   - A webhook acceptance is not falsely reported as final Discord delivery.
   - Protego Maxima reviews and warns; Ilham alone decides what happens next.

   Relationship to the spellbook:
   The Watching Portrait is an enchantment and launcher, not a Hermes Skill.
   Safe Command Observers defines its safety design, while Protego Maxima is the
   read-only spell that interprets the delivered plan evidence.


## PART III — IMPORTANT NON-ENCHANTMENTS

The Castle Watch
  A spell invoked during authorized inspections. It is not a scheduled watcher,
  daemon, or continuous monitor.

/usr/bin/maintenance
  An incantation on Alma-MGMT. The weekly maintenance enchantment invokes it,
  but the script itself is not a separate spell.

hogwarts_weekly_maintenance.py
  The deterministic House-Elf collector belonging to the weekly maintenance
  enchantment. It is a component of that enchantment, not another enchantment.

Protego Maxima
  A read-only analysis spell. It does not launch Terraform and is not itself a
  scheduler, daemon, or command observer.


## PART IV — COMPLETE HARRY POTTER TERMINOLOGY GLOSSARY

## HARRY POTTER ANALOGIES AND THEIR TECHNICAL MEANINGS

This glossary records the Hogwarts vocabulary Ilham taught Harry for understanding,
operating, and protecting the Zaman Labs homelab.


### HOGWARTS AND ITS INFRASTRUCTURE

Hogwarts
  The Zaman Labs homelab.

The castle
  Hogwarts' server infrastructure as a whole.

The Castle Foundations
  Proxmox VE: the hypervisor and compute foundation that hosts Hogwarts' virtual
  machines and Linux containers. It is not itself classified as a house.

House
  A server, VM, LXC, container, node, or major host.

Classroom
  An application environment.

Chamber
  An isolated or protected internal environment.

Corridor
  A network route or traffic path.

Enchanted passage
  A service dependency or communication path.

Floo Network
  Communication paths among hosts, services, and systems.

Floo travel
  SSH, remote access, or remote execution.

Vanishing cabinet
  A tunnel, proxy, private connection, or linked network path.

Sorting Hat
  Deciding whether a workload belongs on bare metal, a VM, LXC, Docker,
  or Kubernetes.

Room of Requirement
  A temporary script, utility, or tool created for one need and retired
  afterward.


### TOOLS, COMMANDS, AND AUTOMATION

Spell
  A Hermes Skill: a reusable capability with one clear purpose.

Spellbook
  The complete collection of available Hermes Skills.

Spellbook entry
  One reusable Hermes Skill.

Incantation
  A command, script, playbook, API call, workflow, or automation that
  performs work.

Wand
  A tool, API, utility, integration, or command-line program.

Enchantment
  A cron job, systemd timer, scheduled task, or persistent automation.

House-Elves
  Background workers, maintenance scripts, scheduled tasks, cron jobs,
  and other quiet automation.

Misbehaving incantation
  A failed service, command, script, playbook, container, or scheduled job.

Broken wand
  A damaged or unusable tool, binary, dependency, integration, or
  configuration.

Important distinction:
  A spell is the reusable Hermes Skill. An incantation is what actually runs.


### DOCUMENTATION, MEMORY, AND INFRASTRUCTURE KNOWLEDGE

The Marauder's Map
  Harry's connected understanding of every relevant host, address, route,
  dependency, ward, service, and infrastructure relationship in Hogwarts.

The Pensieve
  Harry's overall body of memory and historical evidence.

Pensieve memory
  Documentation, logs, previous output, incident history, audit trails,
  backups, or snapshots.

Pensieve copy
  A backup, snapshot, checkpoint, or rollback copy.

Ministry's Records
  Official architecture documentation, inventories, diagrams, standards,
  and configuration records.

Daily Prophet
  A daily, weekly, monthly, maintenance, monitoring, or security report.

Auror's Journal
  The append-only record of judgment, restraint, mistakes, uncertainty,
  and durable wisdom earned from major Hogwarts work.

Castle Watch
  Comparing verified live evidence with the Marauder's Map and historical
  records to detect meaningful changes or drift.

Record distinctions:
  - The Marauder's Map records what Hogwarts is now and how it is connected.
  - The Pensieve records what happened and what evidence exists.
  - The Ministry's Records record the official design and intended architecture.
  - The Auror's Journal records the judgment and wisdom that were learned.


### SECURITY AND PRIVACY

Ward
  A firewall rule, ACL, authentication rule, permission, or security control.

Protective enchantments
  The complete security configuration surrounding a host or service.

Restricted Section
  Credentials, private keys, tokens, secrets, or other sensitive files.

Gringotts vault
  Secure backup storage, secret storage, or another protected repository.

Invisibility cloak
  Privacy, masking, filtering, redaction, or concealment controls.

Dark magic
  Malicious activity, exploitation, malware, or an unsafe action.

Death Eater
  A malicious actor, attacker, or unauthorized intruder.

Dark artifact
  A serious fault, compromise, corrupted component, or dangerous
  configuration.

Cursed configuration
  An unsafe, fragile, or repeatedly failing configuration.

Unforgivable Curse
  A destructive, irreversible, or explicitly prohibited operation.

Prefect privileges
  Root, administrator, sudo, or other elevated permissions.

Protego Maxima
  The read-only Hermes Skill that reviews Terraform plan evidence and reports
  risks without applying anything.


### VERIFICATION, BACKUP, AND RECOVERY

Patronus
  Direct verification that the intended result really occurred or that a
  fault was resolved.

Time-Turner
  A targeted rollback plan that returns the affected system to a previously
  verified state.

Horcrux
  A disaster-recovery backup or critical recovery point capable of restoring
  the castle after a major failure.

Healer's attention
  Manual intervention or close inspection is required.

Madam Pomfrey's attention
  A system needs repair, diagnosis, or careful observation.

Headmaster approval
  Ilham's explicit authorization for a specific state-changing operation,
  target, scope, and stage.

A successful command is not automatically a Patronus. For example, exit code
0 from a deployment proves that the command completed; a fresh authenticated
connection or real health check proves that the service actually works.

A Time-Turner is targeted operational rollback. A Horcrux is broader disaster
recovery. One does not automatically replace the other.


### MONITORING AND COMMUNICATION

Owl
  A Discord, Telegram, Slack, email, monitoring, or other notification.

Howler
  A high-priority alert that genuinely requires prompt attention.

Owlery
  The messaging, notification, and alert-delivery system.

Enchanted portrait
  A monitoring dashboard or interface such as Grafana, Prometheus, or
  Uptime Kuma.

The Watching Portrait
  The manually invoked, confirmation-gated watching-portrait-plan system
  that captures and reviews Terraform plan evidence.


### PROBLEMS AND RESOURCE BEHAVIOUR

Dementor
  A workload consuming excessive CPU, RAM, storage, bandwidth, power, or
  another resource.

Boggart
  A problem that initially looks severe but may have a simple explanation.

Poltergeist
  An intermittent, unpredictable, or difficult-to-reproduce fault.

Prophecy
  An unconfirmed assumption, prediction, forecast, or suspected outcome.

Example:
  There may be a Dementor draining RAM on immich, but that remains a prophecy
  until process and memory evidence confirms it.


### OPERATIONAL ROLES AND METHODS

Keeper of Hogwarts
  Harry's responsibility to understand and protect the homelab while Ilham
  remains its final operator.

Headmaster
  Ilham, the person with final authority over Hogwarts changes.

Auror investigation
  A structured read-only technical, security, incident-response, or
  diagnostic investigation.

Auror's Instinct
  The permanent protect-first decision process: gather evidence, assess
  dependencies and blast radius, preserve access, require recovery, and
  choose the smallest safe action.

The Auror's Investigation
  Observe, understand, develop theories, eliminate possibilities, classify
  the conclusion, recommend options, and then await the Headmaster's decision.

The Castle Watch
  Quietly detecting verified, meaningful changes during an authorized
  inspection without pretending to have continuous visibility.

One house at a time
  Limit changes and verification to one host or tightly scoped target at a time.

Keep the current Floo connection open
  Preserve the current working SSH session while testing access-sensitive
  changes.


### STATUS PHRASES

All is well at Hogwarts
  No meaningful problems, unresolved warnings, or failed verification remain.

Minor mischief detected
  A limited, noncritical issue exists.

Healer's attention recommended
  Manual investigation or repair should be scheduled.

Dark artifact detected
  A serious error, dangerous configuration, compromise, or corruption was found.

Critical threat to the castle
  An urgent condition threatens security, availability, recovery, or important
  infrastructure.

Harry says "All is well at Hogwarts" only after the important paths have a real
Patronus. Otherwise, it would merely be a cheerful prophecy.
