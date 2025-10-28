
這份文件包含：

* 完整架構說明
* DNS 設定
* workflow 範例與環境變數
* 問題排解（403、環境保護、CNAME 清空）

---

# 🌐 GitHub Pages 多子網域自動部署使用說明書

> 適用範例：
> `wedding-front.kuanlin.pro`、`taskgo.kuanlin.pro`、`eric-site.kuanlin.pro`

---

## 🧭 一、整體架構說明

每個子網站（例如 `wedding-front`、`taskgo`）各自是一個 GitHub Repository。
主域名為 `kuanlin.pro`，由 Namecheap 管理。

每次 push 到 `main` 分支時，GitHub Actions 會：

1. 自動 build 專案（Vite / React / Tailwind）
2. 產生靜態檔於 `dist/`
3. 自動寫入正確的 `CNAME`
4. 自動推上 `gh-pages` 分支
5. 自動部署到 `https://<repo-name>.kuanlin.pro`

---

## ⚙️ 二、DNS 設定（Namecheap）

只要設定一次即可讓所有子網域運作。

| Type      | Host | Value                                                                    | TTL       |
| --------- | ---- | ------------------------------------------------------------------------ | --------- |
| **A**     | @    | 185.199.108.153<br>185.199.109.153<br>185.199.110.153<br>185.199.111.153 | Automatic |
| **CNAME** | *    | `and910805.github.io.`                                                   | Automatic |

✅ 這樣所有子網域（例如 `*.kuanlin.pro`）都會指向 GitHub Pages。

---

## 🧩 三、Workflow 設定

每個網站專案都要有這個檔案：

```
.github/workflows/pages.yml
```

內容如下：

```yaml
name: Deploy site to GitHub Pages

on:
  push:
    branches:
      - main
      - 'codex/**'
      - 'dev/**'
  workflow_dispatch:

permissions:
  contents: write

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    env:
      BASE_DOMAIN: kuanlin.pro
      SUBDOMAIN: ${{ github.event.repository.name }}
    steps:
      - name: Checkout source
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install dependencies
        run: npm ci

      - name: Build project
        run: npm run build

      # ✅ 自動生成 CNAME
      - name: Generate CNAME automatically
        run: |
          DOMAIN="${SUBDOMAIN}.${BASE_DOMAIN}"
          echo "🌐 Using domain: $DOMAIN"
          echo "$DOMAIN" > ./dist/CNAME

      # 🚀 Deploy to gh-pages 分支
      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v4
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./dist
          publish_branch: gh-pages


```

---

## 🧠 四、自動偵測子網域邏輯

| 條件                        | 最終產生的 CNAME                 |
| ------------------------- | --------------------------- |
| Repo 名稱 = `wedding-front` | `wedding-front.kuanlin.pro` |
| Repo 名稱 = `taskgo`        | `taskgo.kuanlin.pro`        |
| Repo 名稱 = `eric-site`     | `eric-site.kuanlin.pro`     |

---

## 🛠️ 五、手動覆寫網域（自訂變數）

若要指定不同子網域（例如 `main.kuanlin.pro`），
可以在 GitHub 設定環境變數：

> 路徑：`Settings → Actions → Variables → New repository variable`

| Name            | Value              |
| --------------- | ------------------ |
| `CUSTOM_DOMAIN` | `main.kuanlin.pro` |

然後修改 workflow 的 CNAME 區塊如下：

```yaml
- name: Generate CNAME automatically
  run: |
    DOMAIN="${{ vars.CUSTOM_DOMAIN || github.event.repository.name }}.${BASE_DOMAIN}"
    echo "🌐 Using domain: $DOMAIN"
    echo "$DOMAIN" > ./dist/CNAME
```

---

## 🧱 六、檢查點與驗證

1️⃣ 推上 main 分支：

```bash
git add .
git commit -m "deploy: auto update"
git push
```

2️⃣ 檢查 Actions 頁面
`https://github.com/<user>/<repo>/actions`

3️⃣ 等待顯示：

```
✅ Deploy to GitHub Pages — completed successfully
```

4️⃣ 前往「Settings → Pages」
應顯示：

```
Your site is published at https://<repo>.kuanlin.pro
```

---

## 🧰 七、常見問題排解

| 問題                                           | 原因                     | 解法                                                                                       |
| -------------------------------------------- | ---------------------- | ---------------------------------------------------------------------------------------- |
| ❌ `Permission denied to github-actions[bot]` | workflow 沒有寫入權限        | 到 `Settings → Actions → General → Workflow permissions` → 勾選「Read and write permissions」 |
| ⚠️ `Branch "main" is not allowed to deploy`  | Pages environment 保護限制 | 到 `Settings → Environments → github-pages` → 加入 `main` 或改 workflow 推送至 `gh-pages`        |
| 🕓 `CNAME 被清空`                               | workflow 沒寫入 CNAME     | 確保有 `echo "xxx.kuanlin.pro" > ./dist/CNAME` 這行                                           |
| 🌍 DNS 沒生效                                   | Namecheap 未設定 wildcard | 確認有 `CNAME * and910805.github.io.`                                                       |

---

## 🚀 八、快速複製流程（建立新網站）

1️⃣ 建新 repo
例如 `taskgo`

2️⃣ Push Vite / React 專案程式碼

```bash
git init
git remote add origin https://github.com/and910805/taskgo.git
git branch -M main
git push -u origin main
```

3️⃣ 建立 `.github/workflows/pages.yml`
複製本文件提供的版本

4️⃣ 完成後，系統會自動部署到
`https://taskgo.kuanlin.pro`

---

## 🧹 九、建議補強（進階）

可在 workflow 中加入以下段落清理舊分支：

```yaml
- name: Clean old gh-pages branch before deploy
  run: |
    git fetch origin gh-pages || true
    git branch -D gh-pages || true
```

---

## ✅ 十、檢查部署結果

成功部署後，會看到：

```
✔ build project
✔ generate CNAME
✔ deploy to GitHub Pages
🌐 Using domain: wedding-front.kuanlin.pro
🎉 Deployment successful!
```

---

## 📚 備註

* 所有網站會共用 `kuanlin.pro` 網域下的 DNS 設定
* `gh-pages` 分支自動產生，不需人工管理
* 推 main 就等於「自動上線」

---

### ✨ 作者：Eric (莊冠霖)

Forcera Materials — Dev / InfoSec / Web Engineer
`https://kuanlin.pro`

---

是否要我幫你直接匯出成 `deploy-guide.md` 文件（我可以產生一個下載連結給你）？
或你想加上你的公司標誌（Forcera）與 YAML 自動模板附錄版本？
