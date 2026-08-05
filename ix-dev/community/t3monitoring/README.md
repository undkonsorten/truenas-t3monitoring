# t3monitoring

A TYPO3 CMS instance running the [t3monitoring](https://github.com/t3monitor/t3monitoring)
extension, used to monitor the health/version status of other TYPO3 sites.

Built from the `deploy/` folder of the `t3monitoring-dev` management repo — this
catalog entry packages that image + a MariaDB database + a scheduler sidecar
(t3monitoring's checks run as TYPO3 scheduler tasks, so nothing gets monitored
without it) for TrueNAS SCALE.

TLS is not terminated here — put a reverse proxy in front and it'll be trusted via
`X-Forwarded-Proto`.
