# Active Directory Lab: Domain Services on Hyper-V with Server Core

Built a working Active Directory domain from scratch on Hyper-V: a headless Windows Server 2022 Core domain controller, a domain-joined Windows 11 client, and remote administration with RSAT. Covers the full stack a Tier 1 / junior sysadmin role touches: domain setup, DNS, OUs, users, groups, Group Policy, a Fine-Grained Password Policy, and everyday help desk tasks.

## Objective

Stand up a real Windows domain and run it end to end, to build and evidence the Active Directory skills that IT support and SOC roles expect. I wanted to do it the way it is done in production rather than the easy way: a GUI-less Server Core DC managed remotely from a client, not a point-and-click server with a desktop. The lab also had to fit an 8 GB host, which shaped several of the design choices below.

## Environment

| Component | Details |
|---|---|
| Host | Laptop, 8 GB RAM, Hyper-V |
| Domain Controller | Windows Server 2022 Standard **Core** (no desktop), 2 GB RAM, 30 GB dynamic disk |
| Client | Windows 11, domain-joined, runs RSAT to manage the DC. Started at 4 GB RAM, downgraded to 3 GB (see below) |
| Domain | `lab.local` (NetBIOS `LAB`) |
| Networks | Two Hyper-V virtual switches: an **Internal** switch for isolated lab traffic, and an **External** switch for internet access on the client |
| Addressing | DC `10.0.0.10` (static), client `10.0.0.20` (static, on the Internal switch) |
| Admin tooling | PowerShell on the DC, RSAT (ADUC, Group Policy Management) on the client |

Server Core was a deliberate choice: it runs in about 2 GB instead of the 4 GB the full desktop wants, which is what made two VMs viable on an 8 GB host. The trade-off is no local GUI, so all server administration happened either in PowerShell on the DC or through RSAT from the client, which is how real servers are managed.

Even with Core, 8 GB was tight. With the host OS taking roughly 2.5 GB, the 2 GB DC, and the Windows 11 client at its original 4 GB, the host ran out of physical memory and started swapping heavily whenever the domain and client were both up, which is required for domain join, RSAT management, and policy testing. I downgraded the client from 4 GB to 3 GB to keep both VMs running at once without the host grinding to a halt. The client was slower at 3 GB but usable, and it kept the full lab workflow (DC plus client plus RSAT consoles) within the physical memory available.

## Build

### 1. Isolated network and the Server Core DC

Created a Hyper-V Internal virtual switch so the VMs could talk to each other and the host but reach nothing on my real network. Built the DC as a Generation 2 VM and installed Windows Server 2022 Standard (Core, not Desktop Experience). Server Core boots into SConfig, a small text menu. I typed `15` (Exit to command line) to drop into PowerShell, where the rest of the DC setup was done.

![Server Core SConfig before rename](images/01-sconfig-default-name.png)

Renamed the server to `DC01` and gave it a static IP, since a domain controller must never use a changing address. DNS was pointed at itself, because it was about to become the domain's DNS server.

![DC01 after rename in SConfig](images/02-sconfig-renamed-dc01.png)
![Static IP on the DC](images/03-dc-static-ip.png)

### 2. Promote to domain controller

Installed the AD DS role and promoted the server to a new forest, `lab.local`. This also installed and configured DNS. `Get-ADDomain` confirmed the domain was live.

![Get-ADDomain confirming lab.local](images/04-get-addomain.png)

### 3. Join the Windows 11 client

This is where the first real problem appeared (see Troubleshooting). Once DNS and addressing were correct, the client resolved the domain and joined it.

![Client DNS and connectivity to the DC](images/06-client-dns-ping-success.png)
![Domain join](images/07-client-domain-join.png)
![Windows 11 domain login screen](images/08-client-login-screen.png)

### 4. Remote administration with RSAT

Installed RSAT on the Windows 11 client, which put the full Active Directory Users and Computers (ADUC) and Group Policy Management consoles on the client while the DC stayed headless. From here, the DC could be administered graphically without ever logging into its console.

![ADUC connected to lab.local](images/09-aduc-default-containers.png)

### 5. OUs, users, and groups

Built a small company structure: Organizational Units for IT, HR, and Sales, a user (Jane Doe / `jdoe`), and a security group (`IT-Staff`). Did this in both PowerShell on the DC and the ADUC GUI on the client.

![Creating OUs and the user in PowerShell](images/10-dc-create-ous-user.png)
![Creating the group and adding the member](images/11-dc-create-group.png)
![The IT OU populated in ADUC](images/12-aduc-it-ou-populated.png)
![IT-Staff group membership](images/13-aduc-itstaff-members.png)

### 6. Group Policy

Created and linked three GPOs on the IT OU, each demonstrating a different kind of control:

- **IT-Password-Policy** (a password GPO, for demonstration of GPO creation and linking)
- **IT-Mapped-Drive** (auto-maps an `S:` drive to a share on the DC at login)
- **IT-Lockdown** (blocks access to Control Panel and Settings)

![Three GPOs linked to the IT OU](images/14-gpm-three-gpos.png)

### 7. Fine-Grained Password Policy (real enforcement)

The OU-linked password GPO is a useful demonstration, but it does not actually govern domain account passwords (see Lessons Learned). To enforce a real password rule, I created a **Fine-Grained Password Policy** and applied it to the `IT-Staff` group: 12-character minimum, complexity required, and a 5-attempt lockout threshold. `Get-ADUserResultantPasswordPolicy` confirmed the FGPP governed jdoe.

![FGPP settings and resultant policy for jdoe](images/15-fgpp-settings.png)

I then proved it enforces by attempting to set a weak password, which the policy rejected:

![FGPP rejecting a weak password](images/16-fgpp-reject-weak.png)

### 8. Verifying the policies on the client

Logged in as `jdoe` and confirmed everything applied: the mapped `S:` drive was present, Control Panel was blocked, and `gpresult` showed the two user GPOs.

![Mapped S: drive](images/18-client-mapped-drive.png)
![Control Panel blocked by GPO](images/19-client-control-panel-blocked.png)
![gpresult showing the applied user GPOs](images/20-client-gpresult-jdoe.png)

### 9. Help desk tasks

Ran the everyday help desk tasks: a password reset, an account lockout and unlock, and moving a user between OUs. First, signed in to the client as the domain user `LAB\jdoe` to confirm the account worked end to end.

![Signing in as the domain user LAB\jdoe](images/21b-client-login-jdoe-signin.png)
![jdoe logged in on the client](images/21-client-login-jdoe.png)

**Account lockout and unlock.** Triggered a lockout by failing the sign-in repeatedly. The client first throttled the attempts, then locked the account once the FGPP's 5-attempt threshold was hit.

![Repeated failed sign-ins being throttled](images/22b-client-delaying-attempt.png)
![Account locked out after failed logins](images/22-client-lockout-message.png)

Confirmed the lock on the DC, unlocked the account, and confirmed it was clear again. The combined view below shows the lockout check, the unlock, and the follow-on password reset in one session.

![Unlocking the account on the DC](images/23-dc-unlock-account.png)
![Combined lockout check, unlock, and password reset](images/23b-dc-lockout-reset-combined.png)

**Password reset.** Reset jdoe's password and confirmed the change with `PasswordLastSet`.

![Password reset confirmed](images/17-dc-password-reset.png)

**Logon auditing and OU move.** Confirmed successful logons in the Security log, then moved the user between OUs.

![Successful logon events (4624) on the DC](images/24-dc-logon-events-4624.png)
![Moving the user to the Sales OU](images/25-dc-move-user.png)
![The user now in the Sales OU](images/26-aduc-jdoe-in-sales.png)

## Issues and Troubleshooting

### Client could not reach the DC (no DHCP on an isolated switch)

**Symptom:** the client could not ping the DC, and the domain join failed.

**Cause:** a Hyper-V Internal switch has no DHCP server, so the client gave itself a `169.254.x.x` link-local address that could not route to the DC's `10.0.0.10`.

**Resolution:** assigned the client a static IP (`10.0.0.20`) in the same subnet, with DNS pointed at the DC. After that, `ping` and `nslookup lab.local` both succeeded.

![The failed state before fixing addressing](images/05-client-dns-ping-fail.png)

### Client lost internet after setting the static lab IP

**Symptom:** with the lab IP set, the client could not reach the internet, and moving it back to the external switch left it stuck on the dead lab settings.

**Cause:** a single adapter cannot serve both an isolated static lab network and DHCP internet at once.

**Resolution:** gave the client two adapters, one on the external switch using DHCP for internet and one on the Internal switch using the static lab IP, with a default gateway only on the internet adapter.

### RSAT appeared to install but did nothing

**Symptom:** the Group Policy tools installed, but ADUC was missing and `dsa.msc` would not open.

**Cause:** a typo in the capability name (a `.` where a `-` belonged) meant the command targeted a capability that does not exist, so it installed nothing without a clear error.

**Resolution:** corrected the name, confirmed with `Get-WindowsCapability` that the state read `Installed`, after which ADUC opened and connected to the domain.

### Multi-line commands broke on the Server Core console

**Symptom:** a long user-creation command split into pieces and errored on each line, with PowerShell treating each fragment as its own command.

**Cause:** I tried to break the command across multiple lines in the Core console (pressing Ctrl+Enter or similar to move to the next line), but the console did not continue the command the way I expected, so each line ran on its own and the parameters became unrecognized.

**Resolution:** ran each command as a single line instead. That worked reliably, and single-line commands are now how I work in the Core console.

### GPOs did not apply to the user

**Symptom:** the mapped drive and Control Panel policies did not reach jdoe at first.

**Cause:** jdoe was a member of the IT-Staff group but her user object was not located in the IT OU. Group Policy applies based on the object's OU location, not group membership.

**Resolution:** moved the user object into the IT OU, after which the user-scoped GPOs applied.

### Host ran out of memory with both VMs up

**Symptom:** when the DC and client were both running (needed for domain join, RSAT management, and policy testing), the host became very slow and unresponsive.

**Cause:** 8 GB of physical RAM could not cover the host OS (~2.5 GB), the 2 GB DC, and the 4 GB client at the same time, so the host paged to disk constantly.

**Resolution:** reduced the Windows 11 client from 4 GB to 3 GB. That brought total demand back within physical memory and kept both VMs usable together. The client was slower at 3 GB but workable for the lab tasks.

## Lessons Learned

- **DNS is the linchpin of a domain.** Almost every domain-join failure traces back to the client not using the DC for DNS. Getting `nslookup lab.local` to resolve before attempting the join saves a lot of guesswork.
- **OU location, not group membership, decides what Group Policy applies.** This is a common misconception. Groups control permissions; OUs control policy. A user can be in the right group and still miss a GPO if the user object sits in the wrong OU.
- **Scope matters: user vs computer, and OU vs domain.** A password policy inside an ordinary GPO is a computer setting that only affects local accounts, and it only reaches objects in the OU it is linked to. Domain account password rules come from the Default Domain Policy or a Fine-Grained Password Policy. This is why the OU-linked password GPO never governed jdoe, and why the FGPP (applied to the IT-Staff group) was the correct tool for real enforcement.
- **Server Core forces good habits.** Managing a headless DC through PowerShell and RSAT is closer to production than clicking around a desktop server, and it kept the whole lab inside an 8 GB budget.
- **Credential hygiene, even in a lab.** Throwaway lab passwords were kept out of the published write-up and redacted from screenshots, because how credentials are handled is itself part of a security skill set.

## Skills Practiced

- Building and promoting a Windows Server 2022 domain controller (AD DS, DNS, new forest)
- Administering a headless Server Core DC via PowerShell
- Remote server administration with RSAT (ADUC, Group Policy Management) from a client
- Hyper-V networking: Internal virtual switches, static addressing, multi-adapter design
- Organizational Unit, user, and group management in both PowerShell and GUI
- Group Policy creation, linking, and verification (password policy, drive mapping, lockdown)
- Fine-Grained Password Policy applied to a group, with enforcement proven by a rejected password
- Help desk operations: password reset, account lockout and unlock, moving users between OUs
- Reading security logs (logon 4624, failed logon 4625, lockout 4740)
- Working within hardware constraints: tuning VM memory to run a domain and client on an 8 GB host
