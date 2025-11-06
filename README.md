# Go 安全掃描工具

本文以 **Checkmarx** 作為商業安全掃描的基準工具，探討如何使用 **govulncheck**、**gosec**、**semgrep** 三種開源工具作為補充，實現 **零成本擴展** 和 **早期發現** 的安全保障策略。旨在為開發團隊提供全面的安全工具比較分析和實務應用指南。

---

[功能分析與比較](#功能分析與比較) | [四工具核心比較](#四工具核心比較) | [各工具發現的核心問題 (Checkmarx 基準)](#各工具發現的核心問題-checkmarx-基準) | [實務建議](#實務建議) | [安裝與使用指南](#安裝與使用指南) | [參考資料](#參考資料)

## 功能分析與比較

### **工具定位**
- **govulncheck**: 確保第三方依賴安全
- **gosec**: 確保代碼質量和編程規範
- **semgrep**: 提供廣泛的安全規則覆蓋，支援多種語言，支援自定義規則
- **Checkmarx**: 深度業務邏輯安全分析

### **獨特發現**
- **govulncheck**: 標準庫 CVE (x509, asn1, http)
- **gosec**: Go 編程錯誤 (溢出, 硬編碼, 錯誤處理)
- **semgrep**: 配置問題 (SSL MinVersion, 私鑰檢測)
- **Checkmarx**: 業務邏輯問題 (XSS, 對象綁定, 前端安全)

### **重疊矩陣**
| 比較 | 重疊程度 | 共同發現問題 |
|------|----------|--------------|
| **Checkmarx vs gosec** | ~20% | SSL Verification Bypass |
| **Checkmarx vs semgrep** | ~40% | SSL Bypass, Cookie Security |
| **Checkmarx vs govulncheck** | ~5% | Cookie 解析漏洞 (間接) |
| **三工具間重疊** | ~10-15% | TLS/SSL 安全問題 |

## 四工具核心比較

| 工具 | 問題數量 | 專長領域 | Checkmarx 重疊度 |
|------|----------|----------|------------------|
| **Checkmarx** | 36個 | 應用業務邏輯安全 | 基準 |
| **govulncheck** | 10個 | Go 生態 CVE | ~5% |
| **gosec** | 32個 | Go 編程安全反模式 | ~20% |
| **semgrep** | 5個 | 通用安全規則 | ~40% |

## 各工具發現的核心問題 (Checkmarx 基準)

### **govulncheck (10個問題)**
- **GO-2025-4013**: crypto/x509 DSA 證書驗證崩潰
- **GO-2025-4012**: net/http Cookie 解析內存耗盡
- **GO-2025-4008**: crypto/tls ALPN 錯誤信息泄露
- **其他**: x509, asn1, pem 等標準庫漏洞

### **gosec (32個問題)**
- **G115** (高風險): Integer overflow (9個實例)
- **G402** (高風險): TLS InsecureSkipVerify
- **G407** (高風險): Hardcoded IV
- **G104** (低風險): Errors unhandled (15個實例)
- **G304**: File inclusion (2個實例)

### **semgrep (5個問題)**
- **bypass-tls-verification**: SSL 驗證繞過
- **cookie-missing-secure**: Cookie 缺少 Secure 標誌 (2個實例)
- **missing-ssl-minversion**: SSL 缺少最小版本
- **detected-private-key**: 私鑰文件檢測

### **Checkmarx (36個問題)**
- **高風險**: Reflected XSS All Clients
- **中風險**: Unsafe Object Binding (7個), Cookie 安全 (6個), SSL Bypass 等
- **前端問題**: Client Privacy Violation, Cookie Poisoning

## 實務建議

### **掃描策略**
```bash
# 四重掃描防護網
govulncheck ./...         # 依賴生態安全
gosec ./...               # Go 編程規範
semgrep --config=auto .   # 通用安全實踐
checkmarx scan            # 業務邏輯深度分析
```

## 安裝與使用指南

### **安裝指令**

#### **macOS (使用 Homebrew)**
```bash
# 安裝 Go 安全掃描工具
brew install gosec
brew install govulncheck
brew install semgrep

# 驗證安裝
gosec --version
govulncheck --version
semgrep --version
```

#### **其他系統**
```bash
# gosec
go install github.com/securecodewarrior/gosec/v2/cmd/gosec@latest

# govulncheck (Go 1.21+ 內建)
go install golang.org/x/vuln/cmd/govulncheck@latest

# semgrep
# Ubuntu/Debian
apt-get install semgrep
# 或使用 pip
pip install semgrep
```

### **基本使用**

#### **govulncheck - 依賴漏洞檢查**
```bash
# 檢查當前目錄
govulncheck ./...

# 檢查特定模塊
govulncheck ./path/to/module

# 輸出 JSON 格式
govulncheck -json ./...
```

#### **gosec - Go 編程安全檢查**
```bash
# 基本掃描
gosec ./...

# 指定輸出格式
gosec -fmt=json ./...

# 排除特定規則
gosec -exclude=G101,G102 ./...

# 只檢查特定規則
gosec -include=G401,G501 ./...
```

#### **semgrep - 通用安全規則檢查**
```bash
# 自動配置掃描 (推薦)
semgrep --config=auto .

# OWASP Top 10 規則
semgrep --config=p/owasp-top-ten .

# CWE Top 25 規則
semgrep --config=p/cwe-top-25 .

# 多語言掃描
semgrep --config=auto --lang=go .

# 輸出 SARIF 格式 (CI/CD 整合)
semgrep --config=auto --sarif .
```

### **進階配置**

#### **CI/CD 整合**
```yaml
# GitHub Actions 示例
- name: Security Scan
  run: |
    govulncheck ./...
    gosec -fmt=json -out=gosec-report.json ./...
    semgrep --config=auto --sarif --output=semgrep-report.sarif .
```

#### **自定義規則**
```bash
# semgrep 自定義規則
semgrep --config=auto --config=./custom-rules.yml .

# gosec 自定義規則 (通過代碼註解)
# 在代碼中添加:
// #nosec G101,G102 - 忽略特定規則
func sensitiveFunction() {
    // #nosec - 忽略當前行
}
```

#### **批量處理**
```bash
# 並行執行多個掃描
govulncheck ./... & gosec ./... & semgrep --config=auto . & wait

# 生成綜合報告
./generate-security-report.sh
```

## 參考資料

- 函式庫推薦列表
[https://github.com/Binject/awesome-go-security](https://github.com/Binject/awesome-go-security)

- CWE Top 25 列表
[https://cwe.mitre.org/top25/archive/2024/2024_cwe_top25.html#top25list](https://cwe.mitre.org/top25/archive/2024/2024_cwe_top25.html#top25list)
