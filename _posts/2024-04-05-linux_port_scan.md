---
title : Linux TCP/UDP port scan
date : 2024-04-05 23:00:00 +09:00
categories : [Development, Infra]
tags : [Infra, Linux, Port]
---

## nc 사용

- UDP

    ```bash
    nc -z -w 1 -u {ip} {port}
    ```

- TCP

    ```bash
    nc -z -w 1 {ip} {port}
    ```

## nmap 사용

- UDP(단일)

    ```bash
    nmap -sU -p {port} {IP}
    ```

- UDP(다수)

    ```bash
    nmap -sU -p {port-port} {IP}
    ```

- TCP(단일)

    ```bash
    nmap -sS -p {port} {IP}
    ```

- TCP(다수)

    ```bash
    nmap -sS -p {port-port} {IP}
    ```
