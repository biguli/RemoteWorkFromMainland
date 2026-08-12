# Remote Work from Mainland China: Setup & Tips 🇨🇳💻

A practical guide to seamlessly and securely working remotely from Mainland China on a personal PC.

## 🚀 Overview

Working remotely within Mainland China presents unique challenges, from network routing and localized software ecosystems to data compliance and personal-to-work isolation. This repository provides actionable workflows and battle-tested solutions to optimize your WFH environment.

---

## 🛠️ Case Study: Dual-Proxy Architecture & Routing Optimization

### Environment Setup

- **Host OS:** Windows 11 Home (Chinese)
- **WSL:** Ubuntu (NAT Network Mode)
- **Docker Desktop:** Integrated with default WSL distro
- **Global Proxy (`PROXY`):** VLESS node managed via [`3x-ui`](https://github.com/MHSanaei/3x-ui) on a US VPS (for GitHub and global web traffic)
- **Company VPN (`easyconnect`):** Containerized Sangfor EasyConnect client via [`docker-easyconnect`](https://github.com/docker-easyconnect/docker-easyconnect) providing a local SOCKS5 proxy at `127.0.0.1:1080`
- 
> 📄 **Configuration Reference:** You can check the complete routing configuration example in [`Clash.yaml`](./Clash.yaml).

### 1. Isolated VPN Container Setup

To keep the intrusive enterprise VPN client contained, run EasyConnect inside Docker:

> 💡 **Tip:** If your intranet services require custom corporate domain resolution, pass them directly into the container using `--add-host hostname:ip`.

```bash
docker run --rm \
  --device /dev/net/tun \
  --cap-add NET_ADMIN \
  -ti \
  -e PASSWORD=your_password \
  -e URLWIN=1 \
  -v $HOME/.ecdata:/root \
  -p 127.0.0.1:5901:5901 \
  -p 127.0.0.1:1080:1080 \
  -p 127.0.0.1:8888:8888 \
  --add-host test.company.com:10.x.x.x \
  hagb/docker-easyconnect:7.6.7
```

### 2. Clash Verge (TUN Mode) Rule Setup

Add the following rules to the top of your proxy rule list in Clash Verge:

```yaml
rules:
  - DOMAIN-SUFFIX,docker.com,PROXY
  - PROCESS-NAME,com.docker.backend.exe,DIRECT
  - DOMAIN-SUFFIX,company.com,easyconnect
```

#### Rule Breakdown

- `*.docker.com`: Explicitly routed via `PROXY` so image pulls bypass local restrictions.
- `com.docker.backend.exe`: Strictly set to `DIRECT` to ensure all Docker internal traffic bypasses Clash's TUN interface.
- `*.company.com`: Internal company domains route seamlessly through the local easyconnect SOCKS5 proxy.

### ⚠️ Pitfall & Lesson Learned: Resolving Traffic Loops

#### The Problem

An earlier configuration attempted to bypass the VPN server endpoint while routing other company traffic:

```yaml
# ❌ Flawed Rule Setup
rules:
  - DOMAIN,vpn.company.com,DIRECT
  - DOMAIN-SUFFIX,company.com,easyconnect
```

Under this setup, raw IP traffic or non-standard outbound connections originating from the VPN container were captured back by Clash's TUN interface. This generated a recursive traffic loop (**Traffic Storm**), spawning tens of thousands of active socket connections, exhausting the host's Windows ephemeral port pool, and completely severing network connectivity.

#### The Fix

By configuring the entire Docker backend process (`com.docker.backend.exe`) to `DIRECT`, container traffic bypasses the host TUN interface entirely. This completely eliminates recursive loops while preserving precise split tunneling.

---
## 🎯 Results & Impact
Once configured, all network routes operate concurrently and seamlessly across both the host OS (Windows) and vitrual environments (WSL / Docker containers):

- **Company Intranet:** Directly accessible via containerized EasyConnect proxy.

- **Mainland China Websites & Services:** Directly routed (DIRECT) for maximum speed and ultra-low latency.

- **Global Web & Developer Tools (GitHub, Docker Hub, OpenAI):** Smoothly routed via global proxy node (PROXY).

## 🤝 Contributing

Contributions, local tips, and PRs are welcome! Feel free to open an issue to submit your favorite mainland WFH workflow or hardware setup.

## 📄 License

MIT License
