# Security & Data Privacy assessments

## TRAs, TCPs & BCPs

* [TRAs](https://comcastsec.service-now.com/now/nav/ui/classic/params/target/x_mcse_tra_tra_list.do)
* [TCPs](https://comcastsec.service-now.com/now/nav/ui/classic/params/target/x_mcse_tra_tcp_list.do)
  * [TCP for all XVP services](https://comcastcorp.sharepoint.com/:w:/r/sites/xci-scn/_layouts/15/Doc.aspx?sourcedoc=%7B4751D3F3-F95D-443A-9F51-E4786EDD0C17%7D&file=Comcast%20Cable%20BC-DR%20TCP%20--%20XVP%20Services.docx&action=default&mobileredirect=true)
* [Cheat sheet doc](https://comcastcorp.sharepoint.com/:w:/r/sites/EntertainmentAIandTechPlatforms/_layouts/15/Doc.aspx?sourcedoc=%7BDE5A821F-91E0-4E39-97AC-965D6FBD1C61%7D&file=TRA%20TCP%20cheat%20sheet.docx&action=default&mobileredirect=true) 

## CMDB

Historically, we used one [ITRC ID `39839`](https://itrc.comcast.net/apps/xvpservices) for all XVP services.

### XvpServices

Each XVP service has its own ID but `39839` is used only for shared services

* `docs`
* `config-api`
* `ping-api`

> These three services do not have any customer data. Hence, Hanu_Ramavath@comcast.com marked those application as Out of Focus for IRR Downloads.

### XVP services


* `97620` - [XvpDisco](https://devhub.comcast.net/catalog/default/system/xvpdisco)
* `97627` - [XvpLinear](https://devhub.comcast.net/catalog/default/system/xvplinear)
* `97120` - [XvpPlayback](https://devhub.comcast.net/catalog/default/system/xvpplayback)
* `96829` - [XvpProfile](https://devhub.comcast.net/catalog/default/system/xvpprofile)
* `96829` - [XvpProfile](https://devhub.comcast.net/catalog/default/system/xvpprofile)
* `96787` - [XvpRights](https://devhub.comcast.net/catalog/default/system/xvprights)
* `96836` - [XvpSaved](https://devhub.comcast.net/catalog/default/system/xvpsaved)
* `97108` - [XvpSession](https://devhub.comcast.net/catalog/default/system/xvpsession)
* `97113` - [XvpSports](https://devhub.comcast.net/catalog/default/system/xvpsports)
* `97127` - [XvpTransactional](https://devhub.comcast.net/catalog/default/system/xvptransactional)
* `98180` - [XvpXifa](https://devhub.comcast.net/catalog/default/system/xvpxifa)
* `99493` - [XvpEss](https://devhub.comcast.net/catalog/default/system/xvpess)
* `99496` - [XvpParental](https://devhub.comcast.net/catalog/default/system/xvpparental)
* `99559` - [XvpMindata](https://devhub.comcast.net/catalog/default/system/xvpmindata)

## Threat Model

XVP has gone through a thread model exercise documented as [`TM0004308`](https://comcastsec.service-now.com/now/nav/ui/classic/params/target/u_threat_model.do%3Fsys_id%3Ddd9c602c1b113490b76720a0604bcb44%26sysparm_view%3DCustomer%26sysparm_record_target%3Du_threat_model%26sysparm_record_row%3D1%26sysparm_record_rows%3D1%26sysparm_record_list%3Du_poc%253Db72402c41b707b48107aec217e4bcb86%255EORassigned_to%253Db72402c41b707b48107aec217e4bcb86%255EORopened_by%253Db72402c41b707b48107aec217e4bcb86%255EORadditional_assignee_listCONTAINSb72402c41b707b48107aec217e4bcb86%255EORwatch_listCONTAINSb72402c41b707b48107aec217e4bcb86%255EORu_sdl_team.u_biso%253Db72402c41b707b48107aec217e4bcb86%255EORu_sdl_team.u_team_assigneeCONTAINSb72402c41b707b48107aec217e4bcb86%255EORu_sdl_team.u_team_member_sCONTAINSb72402c41b707b48107aec217e4bcb86%255EORu_sdl_team.u_sdl_poc%253Db72402c41b707b48107aec217e4bcb86%255EORsys_idIN%255EORu_secondary_itrc_id_sIN%255EORu_secondary_itrc_id_sCONTAINS%255EORDERBYDESCsys_created_on).

## PIA

### General

XVP has gone through multiple PIA reviews. The outcome is documented in the [PIA portal](https://comcast.my.onetrust.com/ssp/assessments/list).
Access to a given PIA# is very limited. Reaching out via [PrivacyTeam@comcast.com](mailto:PrivacyTeam@comcast.com) has been successful for any kind of request.

#### PIA #

The PIAs for the XVP platform are:

* Initial Sky UK: `2758`
* Sky IT/DE extension: `5210`
* Foxtel (Australia): `5249`

## XPKI Periodic Certificate Scanning

[XPKI team](https://xpki.io/) requests that all teams monitor their X.509 certificates in **production** use.   They have developed an internal tool to collect/monitor/report/alarm on certificate expiry and TLS versions used by our productions systems.

The XVP team is responsible to run this tool periodically and remediate issues that are discovered.  See [xCaner help page](https://xpki.io/helpPage?path=xCaner) for more information.  You can view current certificate records tracked by this tool by viewing [XPKI's monitor site](https://monitor.xpki.io/pages/xcaner).  XVP's ITRC ID is `39839`.

### Goals

Our scanning efforts support TPX Security goals to:

1. prevent avoidable certificate expiry incidents
2. report/audit/manage the TLS version compliance of exposed production systems

When this scanning is in place, we can expect alarms/notifications to the `xvp_svcs@comcast.com` email when persisted records are near expiration.

### Instructions for running xCaner:

#### Manual xCaner Running:
There are several ways to run the `xCaner` tool ([source](https://github.comcast.com/XH-Security/xcaner)).  Conveniently, a Docker image has been prepared by the XPKI for portability.   At the time of this writing, the latest version (`2.2`) is applied in the Docker tag below.

1. Users who run this tool must be within the `orgEmail` email distro shown below.
2. Collect [FQDNs](https://en.wikipedia.org/wiki/Fully_qualified_domain_name) of all **production** XVP services.
    - include the port number
    - all geo-regions are included in this list.  ie: NA, EU, APAC, etc.
    - an example FQDN could be: `ess.exp.xvp.na-1.xcal.tv:443`
3. run the tool, once per production FQDN.

```bash
docker run -ti hub.comcast.net/xpki/xc:2.2 \
	-t <fqdn:port> \
	-org xvp \
	-orgEmail xvp_svcs@comcast.com \
	-itrc 39839
```

4. Confirm successful scan/logging

You can confirm successful scanning/logging by examining the tooling output.  You should see a `Success` statement.

```text
Success: Scan data attached to the ITRC app: XvpServices and uploaded to the database successfully. ScanId = 28042023142757117257_675445
```
#### Automated xCaner Running:
Xcaner Automation is built using a Bash script and scheduled in a [Concourse pipeline](https://ci.comcast.net/teams/xvp/pipelines/xvp-xcaner-scan).

It utilizes two repositories: the xCaner Repository, which is the security team's repository, and the [SRE Repository](https://github.comcast.com/xvp/sre/tree/main/monitoring_tools/xcaner), which contains scripts.

The authentication process is done using SAT Client and SAT Secret. To perform authentication, the SAT token requires the "x1:xpki:xcaner:auth" capability for both SAT Client and SAT Secret.

The script follows these steps:

1. Install the required packages.
2. Execute the below command to scan the endpoints.

```bash
python3 xcaner.py -i "hosts.txt" -e 3 -org TPX-SoftArch-SDE-AAE_PAPI:NTL -orgEmail xvp_svcs@comcast.com -itrc "$itrcCode" -satClient "$satClient" -satSecret "$satSecret"
```

3. Generate a report in the same location as the script.
4. Move the report to an S3 location for future reference.

Additionally, the python script has been used to send a Slack notification when a certificate is about to expire.
