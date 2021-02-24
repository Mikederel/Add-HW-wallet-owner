---
description: >-
  On Ubuntu/Debian, this guide will illustrate how to install and configure a
  Cardano stake pool from source code on a two node setup with 1 block producer
  node and 1 relay node.
---

# README

Make sure your HW wallet is seen in linux with lsusb command.

If not see: [https://support.ledger.com/hc/en-us/articles/115005165269-Fix-connection-issues](https://support.ledger.com/hc/en-us/articles/115005165269-Fix-connection-issues)

1. Delegate HW wallet to your pool from either Daedalus or Yoroi.

Air-gapped: Install cardano-hw-cli [https://github.com/vacuumlabs/cardano-hw-cli](https://github.com/vacuumlabs/cardano-hw-cli)



{% hint style="info" %}
 test
{% endhint %}

Once you're strong enough, save

## Guide: How to build a Cardano Stake Pool

### 🎉 ∞ Pre-Announcements

{% hint style="info" %}
Thank you for your support and kind messages! It really energizes us to keep creating the best crypto guides. Use [cointr.ee to find our donation ](https://cointr.ee/coincashew)addresses. 🙏
{% endhint %}

{% hint style="success" %}
As of Dec 28 2020, this is **guide version 3.0.0** and written for **cardano mainnet** with **release v.1.24.2** 😁
{% endhint %}

### 🏁 0. Prerequisites

## Add-HW-wallet-owner

First, update packages and install Ubuntu dependencies.

```bash
sudo apt-get update -y
```

```text
sudo apt-get upgrade -y
```

```text
sudo apt-get install git jq bc make automake rsync htop curl build-essential pkg-config libffi-dev libgmp-dev libssl-dev libtinfo-dev libsystemd-dev zlib1g-dev make g++ wget libncursesw5 libtool autoconf -y
```

Install Libsodium.

```bash
mkdir $HOME/git
cd $HOME/git
git clone https://github.com/input-output-hk/libsodium
cd libsodium
git checkout 66f017f1
./autogen.sh
./configure
make
sudo make install
```

Install Cabal.

```bash
cd
wget https://downloads.haskell.org/~cabal/cabal-install-3.2.0.0/cabal-install-3.2.0.0-x86_64-unknown-linux.tar.xz
tar -xf cabal-install-3.2.0.0-x86_64-unknown-linux.tar.xz
rm cabal-install-3.2.0.0-x86_64-unknown-linux.tar.xz cabal.sig
mkdir -p $HOME/.local/bin
mv cabal $HOME/.local/bin/
```

