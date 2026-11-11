# Microsoft Entra ID Lab: Users, Groups, Roles, MFA and Offboarding

A home lab where I set up a small company directory in Microsoft Entra ID (Microsoft's cloud identity service), gave people the right access with groups and roles, saw MFA enforced on a new user, and offboarded a user while reading the sign-in and audit logs to prove it worked.

This is the cloud version of my [Active Directory lab](https://github.com/RushishwarM/active-directory-lab), and it follows my [Microsoft Sentinel lab](https://github.com/RushishwarM/sentinel-siem-lab). In the AD lab I managed identities on a Windows Server. Here I do the same job in the cloud.

## What I built

- A test directory on the **Entra ID Free** tier with 4 lab users (IT, HR, Sales and a contractor)
- 3 security groups mirroring the AD lab departments: `IT-Team`, `HR-Team`, `Sales-Team`
- A **Helpdesk Administrator** role for the IT user (least privilege: password resets, not full admin)
- **Security defaults** turned on, which forces MFA registration for every user
- A first sign-in test that showed MFA being demanded
- Failed sign-ins on purpose, then read them in the sign-in logs
- An **offboarding** test: revoke sessions, disable the account, remove from the group, then check the sign-in log and audit log

## Lab setup

| Item | Detail |
|---|---|
| Tenant | "Default Directory" from my free Azure account, Entra ID Free licence |
| Users | `lab.it01`, `lab.hr01`, `lab.sales01`, `lab.temp01` (job title and department filled in) |
| Groups | `IT-Team` (Lab IT 01), `HR-Team` (Lab HR 01), `Sales-Team` (Lab Sales 01, Lab Temp01) |
| Role | Helpdesk Administrator assigned to Lab IT 01 (directory scope) |
| MFA control | Security defaults enabled |
| Cost | $0. No paid licence or trial used |

![Users list](screenshots/01_users.png)
![Groups](screenshots/02_groups.png)

## Step 1: Users and groups

I created the four users with auto-generated passwords and filled in job title and department, the same attributes an HR system would set. Then I created three security groups with **assigned** membership and added the right people. Group-based access is what real companies use, so permissions follow the group and not the person.

## Step 2: Give only the access that is needed (RBAC)

RBAC (role-based access control) means each person gets only the permissions their job needs. I assigned the built-in **Helpdesk Administrator** role to Lab IT 01. That role can reset passwords for non-admin users, but it is not a Global Administrator.

![Helpdesk Administrator assignment](screenshots/03_helpdesk_role.png)

I also switched off the "Access management for Azure resources" elevated-access setting on my own admin account so it does not keep top-level access it does not need.

## Step 3: MFA with security defaults

MFA (multi-factor authentication) asks for a second proof of identity, such as a code from a phone app, on top of the password. Security defaults is the free way to turn it on for everyone.

![Security defaults enabled](screenshots/04_security_defaults.png)

I signed in as Lab HR 01 in a private window. The first sign-in forced a password change, and the next screen demanded MFA setup ("Let's keep your account secure") with no skip option shown. A user with only a password cannot get in. I did not finish the MFA registration, because the prompt itself is the evidence.

![MFA required](screenshots/05_mfa_required.png)

## Step 4: Failed sign-ins and the sign-in logs

I typed a wrong password three times for Lab Sales 01. Each attempt shows up in the sign-in logs as a failure with error code **50126**, "Error validating credentials due to invalid username or password". The log also shows the other users' successful and interrupted sign-ins during the first-login flow.

![Sign-in log list](screenshots/06_signin_failures_list.png)
![Error 50126](screenshots/07_signin_50126.png)

This is the cloud version of the failed logons (event 4625) I detected in the Sentinel lab.

## Step 5: Offboarding a user (Lab Temp01)

This repeats what my `Disable-LabUser.ps1` script did in the AD lab:

1. Revoked the user's sessions so any existing sign-in stops working
2. Disabled the account
3. Removed the user from `Sales-Team`

![Group member removed](screenshots/08_group_member_removed.png)

Then I tried to sign in as that user. The sign-in page says the account is locked.

![Blocked sign-in](screenshots/09_disabled_user_blocked.png)

The audit log shows the whole sequence with times: the session revoke (15:46:53), **Disable account** (15:47:26) and **Remove member from group** (15:48:19).

![Audit log](screenshots/10_audit_offboarding.png)
![Audit log detail](screenshots/11_audit_detail.png)

The sign-in log gives the real reason as error **50057**, "The user account is disabled."

![Error 50057](screenshots/12_signin_50057.png)

## Things I learned and problems I ran into

**1. The sign-in page message is not the real reason.** The page said "Your account has been locked", but that is the wording Entra shows for a disabled account. The sign-in log (`50057`) is the proof, so I check the log first, not the message.

**2. Conditional Access needs a paid licence.** Conditional Access is the rule engine ("if this user signs in from here, require MFA or block"). It needs Entra ID P1 or P2. The P2 trial asked for payment details and a new account setup, so I did not take it. I used security defaults instead, which are free, and wrote down the policies I would build with a licence (see below).

**3. Logs lag a little.** My offboarding sign-in attempt took a few minutes to show up in the sign-in log. I refreshed and filtered by user to find it.

**4. First sign-in is not just a password.** A brand-new user is forced to change the password and set up MFA before getting anywhere. I did not expect that many steps.

## Designed but not built: Conditional Access policies

These need a P1 or P2 licence, so they are a design and not something I tested:

1. **Require MFA for `HR-Team`.** Start in report-only mode and check the sign-in logs before enforcing. Exclude an emergency admin account so I cannot lock myself out.
2. **Block sign-in from outside Canada.** Use a named location for Canada and block everything else, again report-only first.

## Limits

- Free tier only: no Conditional Access, and limited log history.
- I did not test that Lab IT 01 can reset a password but cannot create users. That is the next check to prove least privilege.
- I did not finish the MFA registration. I only showed that it is required.
- Roles are assigned directly to a user. Assigning them to groups would be cleaner.
- This is a cloud-only directory with 4 test users. It is not connected to my Active Directory lab.

## Skills I practiced

Microsoft Entra ID, users and security groups, RBAC and built-in roles (Helpdesk Administrator), least privilege, MFA and security defaults, sign-in log and audit log analysis (error codes 50126 and 50057), account offboarding (revoke sessions, disable, remove from group), identity lifecycle, free-tier licence limits.

## Planned next

- Test the Helpdesk Administrator role: reset a password (should work) and try to create a user (should be denied)
- Build the two Conditional Access policies in a tenant with a P1 or P2 licence
- Connect this directory to my AD lab with hybrid identity
