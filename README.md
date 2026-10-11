# Reverse DNS Sweep Lab

## Overview
Performed a reverse DNS sweep against a simulated internal network to discover hosts that DNS forward brute-forcing would miss. Reverse DNS uses PTR (Pointer) records to map IP addresses back to hostnames.

## Scenario
Simulated an internal engagement where the only information provided was an IP range (10.10.13.0/24). No hostnames, no service list. The goal was to enumerate DNS PTR records to build a target list.

## Lab Environment
- Scanner: Kali Linux
- Target DNS server: 127.0.0.1 (local dnsmasq instance)
- Target range: 10.10.13.0/24
- Fake domain: insecurecorp.local

### DNS Server Configuration (dnsmasq)
Configured a local dnsmasq instance with the following PTR records to simulate an Active Directory environment:
- 10.10.13.157 -> mail.insecurecorp.local
- 10.10.13.158 -> file.insecurecorp.local
- 10.10.13.159 -> web.insecurecorp.local
- 10.10.13.160 -> chat.insecurecorp.local AND complexname.insecurecorp.local

## Commands Used

### Attempt with dnsrecon
dnsrecon -d placeholder.com -n 127.0.0.1 -t rvl -r 10.10.13.0/24

### Working Alternative: dig + shell loop
echo "=== Reverse DNS Sweep of 10.10.13.0/24 ===" > reverse_output.txt
for i in $(seq 1 254); do
    result=$(dig @127.0.0.1 -x 10.10.13.$i +short 2>/dev/null)
    if [ -n "$result" ]; then
        echo "10.10.13.$i -> $result" >> reverse_output.txt
    fi
done

## Results
10.10.13.157 -> mail.insecurecorp.local.
10.10.13.158 -> file.insecurecorp.local.
10.10.13.159 -> web.insecurecorp.local.
10.10.13.160 -> complexname.insecurecorp.local.
10.10.13.160 -> chat.insecurecorp.local.

## Key Findings
- 5 PTR records across 4 IP addresses discovered.
- **10.10.13.160 hosts TWO hostnames** - complexname and chat. Multiple PTR records on a single IP often indicate a staging or legacy service that has been forgotten about.
- This is exactly the kind of discovery that reverse sweeps are valuable for: finding hosts that a forward brute-force might miss.
- The domain naming convention (insecurecorp.local) reveals the AD forest structure - useful for planning Kerberos attacks.

## Analysis
Reverse DNS sweeps are more valuable than forward brute-forcing when:
1. Only IP ranges are provided (common in internal engagements).
2. Hostnames don't follow predictable patterns.
3. Naming conventions differ between teams or subnets.
4. An attacker wants to find **additional** hosts beyond what forward enumeration reveals.

## dnsrecon -t rvl limitation
dnsrecon's reverse lookup mode did not honor the `-n` custom DNS server argument in this environment, returning 0 records. The `dig` + shell loop approach bypassed this issue and produced clean, reliable results. This is a common pattern: when a purpose-built tool has quirks, drop down to primitive tools (dig, nslookup) and script the loop yourself.

## Key Takeaways
1. Reverse DNS sweeps complement forward enumeration - they reveal hosts that forward tools miss.
2. Multiple PTR records on one IP signal hidden services.
3. Domain naming conventions extracted from PTR records inform downstream attacks (Kerberos, LDAP).
4. When a specialized tool fails, primitive tools + scripting is a viable fallback.

## Evidence
- reverse_output.txt - Raw sweep output

## Tools Used
- dnsrecon
- dig
- dnsmasq (local test DNS server)
