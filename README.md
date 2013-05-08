
ABOUT
=====

`fluxaaa` -- Flux Allocation and Account Adviser

Reports on which Moab allocations user accounts should be added to on the Flux High Performance Compute cluster at the University of Michigan.

Full documentation is available to Flux Support Staff at the University of Michigan at [https://sites.google.com/a/umich.edu/flux-support/support-staff-documents/allocation-requests/flux-allocation-and-account-adviser-script](https://sites.google.com/a/umich.edu/flux-support/support-staff-documents/allocation-requests/flux-allocation-and-account-adviser-script)   It is hoped that that documentation can be moved to GitHub eventually; in the meantime, if you need more information about `fluxaaa` and do not have access to the above web page, please contact markmont@umich.edu.


SYNOPSIS
========

    fluxaaa [--show] [--create-ticket] [--debug] username1 username2 ...

    fluxaaa [--debug] --audit [--passwd-file=FILE] [allocation1 allocation2 ...]


BACKGROUND
==========


The `fluxaaa` script ("Flux Allocation and Account Adviser") was written to address the problem of users inconsistently being added to the correct Moab allocations when their HPC cluster accounts were created.  Originally, this was not a problem, but with an increasing number of broadly shared allocations (e.g., college-wide and department-wide allocations), it became hard for cluster support staff to tell which allocation(s), if any, a new user should be added to.

The fluxaaa script queries and organization's LDAP directory (MCommunity, at the University of Michigan) and then applies a set of policy rules in order to determine which allocations a specific user -- or users in general -- should be in.  Here is an example of the policy rules for the `lsa_flux` allocation:

A user should be added to the `lsa_flux` allocation user list...

* If they have an affiliation that begins with "College of Lit, Science & Arts" (as indicated by the MCommunity attribute `ou`).
* Or if they are employed by LSA (as indicated by the MCommunity attribute `umichHR`).
* Or if they are enrolled in LSA (as indicated by the MCommunity attribute `umichAAAcadProgram`).
* Or if they are sponsored by an LSA department (as indicated by the MCommunity attribute `umichSponsorshipDetail`).

Additionally, people who do not meet the above criteria can be added to the allocation by setting the `%override_add` hash in `fluxaaa.conf`, and people who do meet the above critera can be banned from the allocation by setting the `%override_remove` hash in `fluxaaa.conf`.

The `fluxaaa` script uses a special LDAP account, which has special access privileges, in order to query the MCommunity LDAP directory.  This special account can query information even if the Private or FERPA flags are set for the user's LDAP entry, and this account also has access to data not available to normal users (including HR, enrollment, and sponsorship data).  It is thus very important that this script not be accessible or readable to anyone except for IT staff who are responsible for supporting the HPC cluster, and staff should only use it for processing specific account creation requests or managing allocations.

Currently, policies for allocations are hard-coded into the `fluxaaa` script, in the subroutine `determine_user_allocation`:

* `lsa_flux` (College allocation for LSA: everyone in LSA)
* `polisci_flux` (Department allocation for Political Science: everyone in Political Science)
* `stats_flux` (Department allocation for Statistics: Only Ph.D. students in Statistics; faculty, post-docs, and others get approved/added manually)

To add policies for other allocations, refer to [Adding New Allocation Policies](#addpolicy), below.


ACCOUNT CREATION MODE
=====================

Account creation mode is the default mode of operation for the `fluxaaa` script.  It answers the question, "what allocations should these users have access to?"   Here is an example:

    [markmont@flux-login1 ~]$ /home2/markmont/accounts/fluxaaa bjensen markmont nobody
    lsa_flux: bjensen,markmont
    example_flux: bjensen
    [markmont@flux-login1 ~]$

In the example above, the `fluxaaa` script applies all of the allocation policies it knows about for three users given and returns the list of allocations that those users should be added to.

The `--create-ticket` option can be used to send an email to automatically open a support ticket for the change.  It is recommended that `fluxaaa` be run without the `--create-ticket` option initially, and then run again with `--create-ticket` only if the results are correct.

The `--show` option causes `fluxaaa` to display everything in the users' LDAP (MCommunity directory) entry.  This can be useful for helping to determine who is responsible for supporting a user, for manually creating accounts, or when coding new allocation policies.

    [markmont@flux-login1 ~]$ /home2/markmont/accounts/fluxaaa --show markmont
    
    ------------------------------------------------------------------------
    dn:uid=markmont,ou=People,dc=umich,dc=edu
    
              messagingURI: markmont@umich.edu@gmail.com
            umichDisplaySN: Montague
        umichPostalAddress: LSA Dean: Info. Technology $ 1112 LSA $ 500 S State Street $ Ann Arbor MI 48109
    umichPostalAddressData: {addr1=LSA Dean: Info. Technology}:{addr2=1112 LSA}:{addr3=500 S State Street}:{city=Ann Arbor}:{state=MI}:{postal=48109}:{nation=UNITED STATES}:{nationCode=USA}
           umichAlumStatus: {campus=UM_ANN-ARBOR}:{degSchool=College of Engineering}:{classYr=XXXX}
                umichTitle: App Programmer/Analyst Sr
         umichNameOfRecord: Mark Montague
            umichInstRoles: AlumniAA
                            RegularStaffAA
                   umichHR: {jobCategory=Staff}:{campus=UM_ANN-ARBOR}:{deptId=172900}:{deptGroup=COLLEGE_OF_LSA}:{deptDescription=LSA Dean: Info. Technology}:{deptGroupDescription=College of Lit, Science & Arts}:{deptVPArea=PRVST_EXC_VP_ACA_AFF}:{jobcode=101904}:{jobFamily=210}:{emplStatus=A}:{regTemp=R}:{supervisorId=XXXXXXXX}:{tenureStatus=NA}
          umichDescription: Member of the LSA IT Advocacy and Research Support team.
             umichAltPhone: (cell phone, urgent matters only) +1 734 717-2422
                    notice: Nothing to see here.
                     ferpa: N
                  entityid: XXXXXXXX
                loginShell: /bin/csh
             homeDirectory: /users/markmont
                 gidNumber: 10
                 uidNumber: 5366
               displayName: Mark Montague
                 homePhone: XXX/XXX-XXXX
                      mail: markmont@umich.edu
                       uid: markmont
                 givenName: Mark
           telephoneNumber: 734/763-7413
                        sn: Montague
                        ou: Alumni
                            LSA Dean: Info. Technology - Faculty and Staff
                            College of Lit, Science & Arts - Faculty and Staff
               objectClass: inetOrgPerson
                            posixAccount
                            umichPerson
                            krbForeignPrincipalAux
                            posixGroup
                            organizationalPerson
                            person
                            ndsLoginProperties
                            top
                            DirXML-EntitlementRecipient
                            DirXML-PasswordSyncStatusUser
                        cn: Mark A Montague
                            Mark Montague
    
    lsa_flux: markmont
    [markmont@flux-login1 ~]$

Finally, the `--debug` option will display voluminous amounts of information about what the script is doing, and may be useful for troubleshooting problems.


ADDING NEW ALLOCATION POLICIES
==============================

<a id="addpolicy"></a> (To be written)


SUPPORT
=======

Please send any questions, feedback, requests, or patches to markmont@umich.edu

Additional resources may be available at [http://github.com/markmont/fluxaaa](http://github.com/markmont/fluxaaa)


LICENSE
=======

`fluxaaa` is Copyright (C) 2013 Regents of The University of Michigan.

`fluxaaa` is free software: you can redistribute it and/or modify it under the terms of the GNU General Public License as published by the Free Software Foundation, either version 3 of the License, or (at your option) any later version.

`fluxaaa` is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU General Public License for more details.

You should have received a copy of the GNU General Public License along with `fluxaaa`.  If not, see [http://www.gnu.org/licenses/](http://www.gnu.org/licenses/)

