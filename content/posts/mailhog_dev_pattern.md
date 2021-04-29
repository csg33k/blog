---
title: "Local Email Development Service"
date: 2021-04-21
draft: false
categories: ["blog"]
tags: ["development", "email"]
authors: ["csgeek"]
---


Whenever I've worked on any application that sends out emails it's always an issue on how to mimic the behavior on my local laptop.  Usually wherever the app is deployed is configured to be able to just 'work'.  aka you can send email by connecting to localhost with no auth.

Now, how do I mimic locally? In the past i've tried setting up postfix etc in a docker stack, or more recently doing an smtp relay using google services.

For reference the previous pattern:

```sh
#!/usr/bin/env bash 
docker run --restart always --name mail \
    -e RELAY_HOST=smtp.gmail.com \
    -e RELAY_PORT=587 \
    -e RELAY_USERNAME=user \
    -e RELAY_PASSWORD=secret \
    -p 25:25 \ 
    -d bytemark/smtp

```

This pattern requires you to enable unsecured application in your google account.

The new pattern I've started using of late is leveraging the wonderful tool called [MailHog](https://github.com/mailhog/MailHog).

To set it up simply add the following to your docker-compose.yml file.

```yaml
  mail:
    image: mailhog/mailhog:v1.0.1
    container_name: mail
    ports:
      - 8025:8025
      - 1025:1025
```

If your application is running in docker you don't need to expose 1025, if you're running your app outside of docker, you need to get to 1025 to send mail.

don't forget to bring it up via `docker-compose up -d mail`

At this point you just need to configure your SMTP settings.

Here's an example snippet that I used for my app.

```yaml
reporting:
  to: [csgeek@anyemail-host.com]
  from: "donotreply@foobar.com"
  subject: "Report for things"
  hostname: localhost
  username:
  password:
  port: 1025
```
Main parts you should note is the hostname is `localhost`, and port is `1025`.

Then once everything is done you can connect to port 8025 and retrieve your email.


![Storage Classes](/images/mailhog_dev_pattern/mailhog_screen.png)


