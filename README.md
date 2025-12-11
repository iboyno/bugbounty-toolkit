# bugbounty-toolkit
cat > bugbounty.sh << 'EOF'
#!/bin/bash

# Bug Bounty Toolkit v2.0
# Complete Automation Suite

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
CYAN='\033[0;36m'
MAGENTA='\033[0;35m'
NC='\033[0m'

print_banner() {
    clear
    echo -e "${CYAN}"
    echo "██████╗ ██╗   ██╗ ██████╗     ██████╗  ██████╗ ██╗   ██╗███╗   ██╗████████╗██╗   ██╗"
    echo "██╔══██╗██║   ██║██╔════╝     ██╔══██╗██╔═══██╗██║   ██║████╗  ██║╚══██╔══╝╚██╗ ██╔╝"
    echo "██████╔╝██║   ██║██║  ███╗    ██████╔╝██║   ██║██║   ██║██╔██╗ ██║   ██║    ╚████╔╝ "
    echo "██╔══██╗██║   ██║██║   ██║    ██╔══██╗██║   ██║██║   ██║██║╚██╗██║   ██║     ╚██╔╝  "
    echo "██████╔╝╚██████╔╝╚██████╔╝    ██████╔╝╚██████╔╝╚██████╔╝██║ ╚████║   ██║      ██║   "
    echo "╚═════╝  ╚═════╝  ╚═════╝     ╚═════╝  ╚═════╝  ╚═════╝ ╚═╝  ╚═══╝   ╚═╝      ╚═╝   "
    echo -e "${BLUE}                          Automated Bug Hunting Toolkit${NC}"
    echo -e "${YELLOW}================================================================================${NC}"
}

check_root() {
    if [[ $EUID -eq 0 ]]; then
        echo -e "${RED}[!] Warning: Running as root is not recommended!${NC}"
        read -p "Continue anyway? (y/N): " -n 1 -r
        echo
        if [[ ! $REPLY =~ ^[Yy]$ ]]; then
            exit 1
        fi
    fi
}

check_deps() {
    echo -e "${YELLOW}[*] Checking dependencies...${NC}"
    
    deps=("curl" "git" "wget" "python3" "nmap" "whois" "dig")
    missing=()
    
    for dep in "${deps[@]}"; do
        if command -v "$dep" &>/dev/null; then
            echo -e "${GREEN}[+] $dep installed${NC}"
        else
            echo -e "${RED}[-] $dep missing${NC}"
            missing+=("$dep")
        fi
    done
    
    if [ ${#missing[@]} -gt 0 ]; then
        echo -e "${YELLOW}[*] Installing missing dependencies...${NC}"
        install_deps
    fi
}

install_deps() {
    if command -v apt &>/dev/null; then
        sudo apt update
        sudo apt install -y "${missing[@]}"
    elif command -v yum &>/dev/null; then
        sudo yum install -y "${missing[@]}"
    elif command -v brew &>/dev/null; then
        brew install "${missing[@]}"
    else
        echo -e "${RED}[!] Can't auto-install. Please install manually:${NC}"
        echo "    ${missing[*]}"
    fi
}

show_menu() {
    echo ""
    echo -e "${MAGENTA}[MAIN MENU]${NC}"
    echo -e "${CYAN}1.${NC} Full Target Reconnaissance"
    echo -e "${CYAN}2.${NC} Quick Vulnerability Scan"
    echo -e "${CYAN}3.${NC} Subdomain Enumeration"
    echo -e "${CYAN}4.${NC} Port Scanning"
    echo -e "${CYAN}5.${NC} Web Technology Detection"
    echo -e "${CYAN}6.${NC} Generate Report"
    echo -e "${CYAN}7.${NC} Install/Update Tools"
    echo -e "${CYAN}8.${NC} View Previous Results"
    echo -e "${CYAN}9.${NC} Settings"
    echo -e "${RED}0.${NC} Exit"
    echo ""
}

recon_full() {
    echo -e "${YELLOW}[*] Enter target domain:${NC}"
    read -r target
    echo -e "${GREEN}[+] Starting full recon on: $target${NC}"
    
    mkdir -p "results/$target"
    
    # WHOIS
    echo -e "${BLUE}[*] WHOIS lookup...${NC}"
    whois "$target" > "results/$target/whois.txt"
    
    # DNS
    echo -e "${BLUE}[*] DNS enumeration...${NC}"
    {
        echo "=== A Records ==="
        dig "$target" A +short
        echo ""
        echo "=== AAAA Records ==="
        dig "$target" AAAA +short
        echo ""
        echo "=== MX Records ==="
        dig "$target" MX +short
        echo ""
        echo "=== TXT Records ==="
        dig "$target" TXT +short
        echo ""
        echo "=== NS Records ==="
        dig "$target" NS +short
    } > "results/$target/dns.txt"
    
    # Nmap
    echo -e "${BLUE}[*] Port scanning...${NC}"
    nmap -sV -sC -T4 "$target" -oN "results/$target/nmap.txt"
    
    # HTTP headers
    echo -e "${BLUE}[*] Checking HTTP headers...${NC}"
    curl -I "http://$target" > "results/$target/http_headers.txt" 2>/dev/null || true
    curl -I "https://$target" > "results/$target/https_headers.txt" 2>/dev/null || true
    
    echo -e "${GREEN}[+] Recon complete! Results saved in results/$target/${NC}"
}

subdomain_enum() {
    echo -e "${YELLOW}[*] Enter target domain:${NC}"
    read -r target
    
    echo -e "${GREEN}[+] Enumerating subdomains for: $target${NC}"
    
    # Using common wordlist
    echo -e "${BLUE}[*] Using built-in wordlist...${NC}"
    subdomains=("www" "mail" "ftp" "blog" "dev" "test" "admin" "portal" "api" "mobile")
    
    for sub in "${subdomains[@]}"; do
        full="$sub.$target"
        if host "$full" &>/dev/null; then
            echo -e "${GREEN}[+] Found: $full${NC}"
            echo "$full" >> "results/$target/subdomains.txt"
        fi
    done
    
    echo -e "${GREEN}[+] Subdomain scan complete!${NC}"
}

port_scan() {
    echo -e "${YELLOW}[*] Enter target IP/Domain:${NC}"
    read -r target
    
    echo -e "${GREEN}[+] Scanning ports for: $target${NC}"
    
    echo -e "${BLUE}[*] Quick scan (top 100 ports)...${NC}"
    nmap -T4 -F "$target" | grep "open"
    
    echo -e "${BLUE}[*] Service detection...${NC}"
    nmap -sV "$target" -p 80,443,22,21,25,3306,3389 | grep open
}

web_tech() {
    echo -e "${YELLOW}[*] Enter target URL:${NC}"
    read -r target
    
    echo -e "${GREEN}[+] Detecting web technologies...${NC}"
    
    # Check whatweb if available
    if command -v whatweb &>/dev/null; then
        whatweb "$target"
    else
        echo -e "${BLUE}[*] Using curl for basic detection...${NC}"
        curl -I "$target" 2>/dev/null | grep -i "server\|powered-by\|x-powered"
    fi
}

generate_report() {
    echo -e "${YELLOW}[*] Enter target name for report:${NC}"
    read -r target
    
    if [ ! -d "results/$target" ]; then
        echo -e "${RED}[!] No results found for $target${NC}"
        return
    fi
    
    report_file="reports/$target-$(date +%Y%m%d).md"
    mkdir -p reports
    
    cat > "$report_file" << EOF
# Bug Bounty Report: $target
**Date:** $(date)
**Generated by:** Bug Bounty Toolkit v2.0

## Executive Summary
Security assessment performed on $target.

## Findings

### 1. DNS Information
\`\`\`
$(cat "results/$target/dns.txt" 2>/dev/null || echo "No DNS data")
\`\`\`

### 2. Open Ports
\`\`\`
$(grep "open" "results/$target/nmap.txt" 2>/dev/null | head -20 || echo "No port scan data")
\`\`\`

### 3. Recommendations
1. Review open ports and services
2. Check for outdated software versions
3. Implement security headers
4. Regular vulnerability scanning

---
*Report generated automatically. Manual verification required.*
EOF
    
    echo -e "${GREEN}[+] Report generated: $report_file${NC}"
}

install_tools() {
    echo -e "${YELLOW}[*] Which tools to install?${NC}"
    echo "1. ProjectDiscovery tools (nuclei, subfinder, httpx)"
    echo "2. SQLMap"
    echo "3. Nikto"
    echo "4. All tools"
    echo "5. Back"
    
    read -r choice
    
    case $choice in
        1)
            echo -e "${BLUE}[*] Installing ProjectDiscovery tools...${NC}"
            # These would be installed via go
            echo "Install manually:"
            echo "go install -v github.com/projectdiscovery/nuclei/v2/cmd/nuclei@latest"
            echo "go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest"
            echo "go install -v github.com/projectdiscovery/httpx/cmd/httpx@latest"
            ;;
        2)
            echo -e "${BLUE}[*] Installing SQLMap...${NC}"
            git clone --depth 1 https://github.com/sqlmapproject/sqlmap.git tools/sqlmap
            ;;
        3)
            echo -e "${BLUE}[*] Installing Nikto...${NC}"
            git clone https://github.com/sullo/nikto.git tools/nikto
            ;;
        4)
            echo -e "${BLUE}[*] Installing all tools...${NC}"
            mkdir -p tools
            ;;
        *)
            return
            ;;
    esac
}

view_results() {
    echo -e "${YELLOW}[*] Available results:${NC}"
    
    if [ -d "results" ]; then
        ls -la results/
        
        echo -e "\n${YELLOW}[*] Enter target to view:${NC}"
        read -r target
        
        if [ -d "results/$target" ]; then
            echo -e "${GREEN}[+] Files for $target:${NC}"
            ls -la "results/$target/"
            
            echo -e "\n${YELLOW}[*] Enter filename to view (or 'all'):${NC}"
            read -r file
            
            if [ "$file" = "all" ]; then
                for f in "results/$target"/*; do
                    echo -e "\n${CYAN}=== $(basename "$f") ===${NC}"
                    head -20 "$f"
                done
            elif [ -f "results/$target/$file" ]; then
                less "results/$target/$file"
            fi
        fi
    else
        echo -e "${RED}[!] No results directory found${NC}"
    fi
}

settings_menu() {
    while true; do
        echo -e "\n${MAGENTA}[SETTINGS]${NC}"
        echo "1. Change output directory"
        echo "2. Configure API keys"
        echo "3. Update wordlists"
        echo "4. Back"
        
        read -r choice
        
        case $choice in
            1)
                echo -e "${YELLOW}[*] Current dir: $(pwd)/results${NC}"
                echo "Enter new directory:"
                read -r new_dir
                echo "export BB_OUTPUT_DIR=\"$new_dir\"" > .bb_config
                ;;
            2)
                echo -e "${BLUE}[*] API keys configuration${NC}"
                echo "Enter Shodan API:"
                read -r shodan_key
                echo "Enter VirusTotal API:"
                read -r vt_key
                echo "SHODAN_KEY=$shodan_key" >> .bb_config
                echo "VT_KEY=$vt_key" >> .bb_config
                ;;
            3)
                echo -e "${BLUE}[*] Updating wordlists...${NC}"
                wget https://raw.githubusercontent.com/danielmiessler/SecLists/master/Discovery/DNS/subdomains-top1million-5000.txt -O wordlists/subdomains.txt
                ;;
            4)
                break
                ;;
        esac
    done
}

main() {
    check_root
    print_banner
    check_deps
    
    while true; do
        show_menu
        echo -e "${YELLOW}[*] Select option:${NC}"
        read -r choice
        
        case $choice in
            1) recon_full ;;
            2) echo "Quick scan function" ;;
            3) subdomain_enum ;;
            4) port_scan ;;
            5) web_tech ;;
            6) generate_report ;;
            7) install_tools ;;
            8) view_results ;;
            9) settings_menu ;;
            0)
                echo -e "${GREEN}[+] Happy Bug Hunting! ${NC}"
                exit 0
                ;;
            *)
                echo -e "${RED}[!] Invalid option${NC}"
                ;;
        esac
        
        echo -e "\n${YELLOW}[*] Press Enter to continue...${NC}"
        read -r
    done
}

# Error handling
trap 'echo -e "\n${RED}[!] Script interrupted${NC}"; exit 1' INT

# Run main function
main "$@"
EOF
