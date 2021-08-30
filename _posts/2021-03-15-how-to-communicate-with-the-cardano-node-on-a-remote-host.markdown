---
layout: post
title:  "How to communicate with the cardano node on a remote host"
author: Srdjan Stankovic 
date:   2021-08-30 17:00:00 +0100
tags: [blockchain,cardano,tcp,aws,devops,docker,nft,marketplace]
image: "/assets/cardano.jpeg"
summary: "Recently, while working on Cardano based NFT Marketplace Cardano Blue, my team and I had to come up with a way for our backend to communicate with the cardano node on a remote host"
---

Recently, while working on [Cardano based NFT Marketplace][cardano-blue] (currently running on testnet), my team and I had to come up with a way for our backend to communicate with the cardano node on a remote host.

Given that our backend is deployed on AWS Fargate we had two choices:
- Deploy cardano-node on AWS Fargate and somehow share UNIX socket file with our backend
- Somehow expose UNIX socket file via some other protocol

We first tried to deploy the whole Cardano graphql stack (including cardano node) on AWS Fargate but this caused so much trouble for us so we decided to go try the other option.
While exploring for possible solutions we stumbled upon socat, a utility that lets you expose your UNIX socket via TCP (and much more honestly). If you are interested in finding more about socat here is an excellent [post][socat-post] that I have personally used to come up with this solution.


## Exposing Cardano Node socket over TCP

To expose cardano node UNIX socket file we first need to install the socat package. Installation will depend on your OS and package manager but for most major package manager socat should be available under the same name.
To install socat on Debian host (or any other OS using apt-get package manager) you can simply type:
sudo apt-get update # update packages
sudo apt-get install socat

Now when we have assured that we have required package (socat), to expose our cardano UNIX node socket path we should run following command:

{% highlight bash %}
socat UNIX-LISTEN:$CARDANO_NODE_SOCKET_PATH,fork,reuseaddr,unlink-early, TCP:127.0.0.1:3333
{% endhighlight %}

This should start socat on localhost and expose our cardano node socket file, whose value is set in the `CARDANO_NODE_SOCKET_PATH` environment variable.
In case you don't have this environment variable set, you can do it by running:

{% highlight bash %}
export CARDANO_NODE_SOCKET_PATH='/path/to/cardano-node/node.socket'
{% endhighlight %}

If you are running a cardano node with Docker you can checkout [my repo][repo] where I have forked cardano-graphql repository and added a Docker container that exposes cardano node socket file via TCP. It's currently on v4.0.0 of cardano-graphql but I might update if there is enough interest for it.

## Connecting to remote cardano node host via TCP

In order to connect to the cardano node on a remote host, we should run following command (given that we have socat already installed on our machine and exposed node socket on other):

{% highlight bash %}
socat UNIX-LISTEN:/path/to/local/node.socket,fork,reuseaddr,unlink-early, TCP:127.0.0.1:3333
{% endhighlight %}

You should replace `UNIX-LISTEN` value to some path on our local machine where we're going to hold our `node.socket` file, and also replace `TCP` value with IP of your remote machine.
If we now set `CARDANO_NODE_SOCKET_PATH` to the destination on our local machine we should be able to use cardano cli without running cardano node on our local machine.

## Conclusion

While we're close to smart contracts on the cardano blockchain we still need a way to communicate with our cardano node over the CLI. Our team at [Cardano Blue][cardano-blue] has taken advantage of cardano-cli and made sure trading NFTs is possible even without smart contracts yet available but once they're out I am sure developing applications like this will be much easier.

[cardano-blue]: http://stage.cardano.blue
[repo]: https://github.com/pyropy/cardano-graphql
[socat-post]: https://www.redhat.com/sysadmin/getting-started-socat