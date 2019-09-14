---
title: 32 things to do before deploying to production
date: 2016-03-01 21:06:35
---

Following the recent trend of *"10 Things You Need To..."* articles I decided to write my own list of things that you need to do. Nah. I have just built the list based on my experience and mistakes that I do not want to repeat. This is my checklist for future production deployments.

<!--more-->

## Production deployment checklist

Considering that each point of the list might be a good topic for a separate article I described each of them really briefly. For some of them I give an example of software that I recommend (there may be better alternatives).

### Deployment automation

- **Staging environment**
	Set up a staging environment which should look the same as production. Automate your deployment and upgrade process. Consider the usage of packages ([sbt-native-packager]), configuration management tools ([chef], [ansible]) and containers ([docker]).
- **Continuous Integration**
	Make your CI ([jenkins], [bamboo]) able to build, test and deploy newest version of the application to the staging environment.

### Monitoring

- **Collect and display metrics**
	Collect usage of CPU, memory, disk, network from all hosts ([collectd]). Collect number of requests per seconds and response times of services ([metrics]). Store data in a central point ([graphite]). Display metrics as graphs ([grafana]).
- **Collect and manage logs**
	Collect, store and analyze logs. It will help you in finding and resolving problems faster. ([splunk], [logstash])
- **Configure alerts**
	Configure monitoring system to send an email to *support* in case of a system outage ([grafana-alerts]).

### Testing

- **Real-life simulations**
	Prepare testing scenarios which simulate real traffic ([gatling]). Run tests all the time (even during upgrades) to make sure that system behaves properly. Monitoring helps a lot during that step.
- **Smoke tests**
	Prepare smoke tests (sanity tests) to verify basic functionality of the system which can be run even on a production environment. It can be used to be notified as soon as possible in case something goes wrong on production.

### App

- **Memory leaks**
	You have an environment, monitoring, and tests. Run them for several hours and check if the memory usage is not increasing during constant load. There are also tools that might help like [plumbr].
- **Blocking threads**
	When running tests, check if there are no unexpected blocking of threads. Tools like [visualvm] or [yourkit] help a lot.
- **Keep alive HTTP connections**
	When using an HTTP client, make sure it is using a pool of keep-alive connections.
- **Confidential data in logs**
	Ensure there are not passwords and other confidential data in the logs.
- **Graceful shutdown**
	On shutdown application should stop accepting new requests and finish the already processing ones.
- **Backpressure**
	If you app is doing some asynchronous work, it might get into a state when is overloaded and not able to handle more tasks. Think about a backpressure mechanism to avoid that.
- **Input validation**
	Ensure proper validation and filtering of incoming data. Prevent SQL injection.
- **Audit logs**
	Log every user action as audit logs.

### DevOps

- **Database indexes**
	Make sure that all required indexes are set. The application probably should not do full table scan queries at all. Disable full table scan queries in database and run tests to verify that.
- **Memory settings**
	Adjust memory settings of services according to host capacity.
- **OS hardening**
	Adjust operating system settings to increase the performance. My two most common tweaks are to increase the maximum number of open file descriptors (*ulimit -n*) and allow reusing sockets (by enabling *tcp_tw_reuse*).
- **Log rotation**
	Avoid full disk by enabling log rotation ([logback]). Backup archived logs if necessary.
- **Startup script**
	Prepare start/stop/restart scripts for the application ([upstart]). It might be useful if the start script waits and checks if the application did actually start properly.
- **Backups, updates, and reverts**
	Schedule backups, test update process and prepare disaster recovery plan.

### UI

- **Tests on different browsers, smartphones, and tablets**
	For that use a smart tool like [browserstack].
- **Input field validation**
	Test if input fields have proper validation attached.
- **HttpOnly cookie**
	[HttpOnly] cookie cannot be accessed from javascript and prevents XSS attacks.
- **CSRF protection**
	Use CSRF tokens to protect users from executing unwanted actions ([csrf]).
- **Minification**
	Reduce the size of js, html and css files and improve the performance using minifiers.

### Security

- **Configure access to hosts**
	Make sure only authorized people can access hosts.
- **Expose only required ports**
	Make only ports for public endpoints open. Bind other services to localhost.
- **HTTPs for public endpoints**
	Use HTTPs and signed SSL certificate on public endpoints.
- **Brute force and DDoS protection**
	Think about brute force and DDoS protection before it is too late.
- **Strong hashing algorithm for passwords**
	Make sure you are not using a weak hashing algorithm for passwords.

### Documentation

- **Prepare/update documentation**
	Remember to finish the documentation before final deployment. For APIs use tools like [swagger].

## Summary

This list for sure is not exhaustive. For each type of application, there are different things that can be easily forgotten. But I think that nowadays most of these things are common for JVM/REST application and can be used for a quick basic check.

[sbt-native-packager]: http://www.scala-sbt.org/sbt-native-packager/index.html
[ansible]: https://www.ansible.com/
[logback]: https://gist.github.com/jcraane/5921329
[chef]: https://www.chef.io/chef/
[puppet]: https://puppetlabs.com/
[browserstack]: https://www.browserstack.com/
[docker]: https://www.docker.com/
[yourkit]: https://www.yourkit.com/
[gatling]: gatling.io
[swagger]: http://swagger.io/
[plumbr]: https://plumbr.eu/
[visualvm]: https://visualvm.java.net/
[collectd]: https://collectd.org/
[metrics]: https://dropwizard.github.io/metrics/3.1.0/
[graphite]: http://graphite.wikidot.com/
[grafana]: http://grafana.org/
[splunk]: http://www.splunk.com/
[csrf]: https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF)
[jenkins]: https://jenkins-ci.org/
[HttpOnly]: https://www.owasp.org/index.php/HttpOnly
[upstart]: http://upstart.ubuntu.com/cookbook
[bamboo]: https://www.atlassian.com/software/bamboo
[grafana-alerts]: https://github.com/pabloa/grafana-alerts
[logstash]: https://www.elastic.co/products/logstash
