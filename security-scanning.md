# Security Scanning & Mitigations

The current process for security scanning and mitigations is as follows:

* XVP code base is integrated with a [static code analysis](../Platform/static-code-analysis.md) tool.
* XVP code base is integrated with [XGitGuard](https://xgitguard.comcast.net/dashboard) to scan for secrets.
* [Prisma Cloud infrastructure via MyApps](https://myapps.microsoft.com/)
* [Snyk Scan](snyk-security-scanning.md)
* xCaner xPKI updates via [`xvp-xcaner-scan` pipeline](https://ci.comcast.net/teams/xvp/pipelines/xvp-xcaner-scan)
* Qualys via [Greenhouse](https://app.securitycentral.comcast.com/#/findings?activeGroupId=60d230f5fbccac2c2a7188d5&activeGroupType=ORGANIZATION&activeGroupLabel=XVP%2520Services&excludeOrganizationChildren=false)
* [SECVULN & VT](https://comcastsec.service-now.com/sn_vul_vulnerable_item_list.do?sysparm_query=cmdb_ci.u_sdl_team.nameSTARTSWITHXVP%20Technical%20Architecture&sysparm_first_row=1&sysparm_view=&sysparm_choice_query_raw=&sysparm_list_header_search=true) as identified by security & third parties 

## Process

1. Review findings from the tools above via [Greenhouse](https://app.securitycentral.comcast.com/#/findings?activeGroupId=60d230f5fbccac2c2a7188d5&activeGroupType=ORGANIZATION&activeGroupLabel=XVP%2520Services&excludeOrganizationChildren=false)
2. Lookup & investigate details of the findings from Greenhouse in their respective tools listed above
3. Include issues of high severity in SCRUM backlog grooming, either as part of Tech Arch, SRE or XVP Stream teams
   * Larger tickets added to quarterly planning & commitments
4. Address issues and verify capture/remediation in Greenhouse
