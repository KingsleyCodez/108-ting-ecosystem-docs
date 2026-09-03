# CI Standard and Workflow Catalog — 108 Ting Ecosystem

**Status:** Proposed standard & implementation catalog (Resolves #16)  
**Applies to:** All 48 repositories in the `108-Plaza` organization  
**Standing Rule:** *"ไม่มีด่าน = ด่านที่พังเงียบ"* (A repository without CI will never signal failure and appears deceptively healthy).

---

## 1. Decision & Policy (ข้อตัดสินตาม Issue #16)

1. **6 repo ที่มีโค้ดจริงทั้งหมดต้องมี CI:**
   ทุก repo ที่มีโค้ดจริงต้องมี automated gate ขั้นต่ำเพื่อตรวจสอบการคอมไพล์/บิลด์, การรัน test, และ linting ห้ามปล่อยให้ repo ใดทำงานโดยไม่มี safety net
2. **Infra Repos (`108-deploy` และ `ting-service-chart`) ต้องมี Lint:**
   สคริปต์ shell และ Helm chart ที่ไม่มีการตรวจไวยากรณ์จะพังตอนนำไปใช้งานจริงบน production เท่านั้น ดังนั้นต้องมี `shellcheck` และ `helm lint`
3. **การส่งมอบงาน (Delivery Strategy):**
   เปิด Issue และส่ง PR แยกราย repo เพื่อให้สามารถรีวิว, ทดสอบ, และ merge ได้อย่างอิสระโดยไม่บล็อกกันเองตามกฎข้อที่ 1 ของ `ENGINEERING_STANDARD_AI_AUTHORED.md` (*"A repo is one atomic, self-verifiable unit of change"*)

---

## 2. Runner Allocation Strategy

ปัญหาหลักในอดีตคือคิวของ self-hosted runner ติดขัดหรือบาง repo ไม่ได้อยู่ใน org runner group ทำให้ workflow ค้างตลอดกาล:

| ประเภท Repo | การตั้งค่า Runner | เหตุผล |
|---|---|---|
| **Public Repos** (เช่น `memory-search`, `pos108-downloads`, `thai-geography`) | `runs-on: ubuntu-latest` | ฟรี, รันได้ทันทีบน GitHub infrastructure ไม่ต้องต่อคิว self-hosted ของ Dell |
| **Private Repos / Repos ทั่วไป** | `runs-on: [self-hosted, Linux, X64]` | รองรับการเข้าถึง registry ภายในและ network ขององค์กร |
| **Monorepo / Complex build** | แยกตาม job (เช่น Rust ใช้ self-hosted / TS-Dart ใช้ ubuntu-latest หรือ self-hosted) | ลดภาระ resource และแยก failure domain |

### Concurrency Block (ทุก workflow ต้องมี)
เพื่อป้องกัน runner ค้างเมื่อมีการ push บ่อยครั้ง:
```yaml
concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

---

## 3. Workflow Catalog ราย Repository

### 3.1. `108Note-Platform`
- **ลักษณะ:** 3 ภาษาใน repo เดียว (`note-api` Rust, `note-ui` TypeScript, `note_app` Flutter/Dart)
- **ไฟล์เป้าหมาย:** `.github/workflows/ci.yml`

```yaml
name: ci

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  note-api:
    name: "note-api (Rust)"
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: clippy, rustfmt
      - uses: Swatinem/rust-cache@v2
        with:
          workspaces: note-api
      - name: Check
        run: cargo check --manifest-path note-api/Cargo.toml
      - name: Clippy
        run: cargo clippy --manifest-path note-api/Cargo.toml -- -D warnings
      - name: Test
        run: cargo test --manifest-path note-api/Cargo.toml

  note-ui:
    name: "note-ui (Next.js / TS)"
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
          cache-dependency-path: note-ui/package-lock.json
      - name: Install dependencies
        run: npm ci --prefix note-ui
      - name: Typecheck
        run: npm run typecheck --prefix note-ui --if-present
      - name: Test
        run: npm test --prefix note-ui --if-present
      - name: Build
        run: npm run build --prefix note-ui

  note-app:
    name: "note_app (Flutter)"
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          channel: 'stable'
          cache: true
      - name: Install dependencies
        working-directory: note_app
        run: flutter pub get
      - name: Analyze
        working-directory: note_app
        run: flutter analyze
      - name: Test
        working-directory: note_app
        run: flutter test
```

---

### 3.2. `memory-search`
- **ลักษณะ:** Rust vector / text memory indexer (Public repo)
- **ไฟล์เป้าหมาย:** `.github/workflows/ci.yml`

```yaml
name: ci

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:
    name: "test · clippy · fmt"
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: clippy, rustfmt
      - uses: Swatinem/rust-cache@v2
      - name: Format check
        run: cargo fmt --all -- --check
      - name: Clippy
        run: cargo clippy --all-targets -- -D warnings
      - name: Test
        run: cargo test --all-targets
```

---

### 3.3. `shop108`
- **ลักษณะ:** Omnichannel commerce bridge (Rust)
- **ไฟล์เป้าหมาย:** `.github/workflows/ci.yml`

```yaml
name: ci

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:
    name: "cargo test"
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: clippy
      - uses: Swatinem/rust-cache@v2
      - name: Check
        run: cargo check
      - name: Clippy
        run: cargo clippy --all-targets -- -D warnings
      - name: Test
        run: cargo test
```

---

### 3.4. `img2img-retouch`
- **ลักษณะ:** Image retouching service (Python)
- **ไฟล์เป้าหมาย:** `.github/workflows/ci.yml`

```yaml
name: ci

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  lint-and-test:
    name: "ruff · pytest"
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
          cache: "pip"
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install ruff pytest
          if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
      - name: Lint with ruff
        run: ruff check .
      - name: Test with pytest
        run: pytest
```

---

### 3.5. `thai-names`
- **ลักษณะ:** Thai name generation and transliteration (Rust & Python)
- **ไฟล์เป้าหมาย:** `.github/workflows/ci.yml`

```yaml
name: ci

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  rust-test:
    name: "cargo test"
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - name: Test
        run: cargo test --if-present

  python-test:
    name: "pytest"
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - name: Test
        run: |
          pip install pytest
          if [ -d tests ] || [ -f test_*.py ]; then pytest; fi
```

---

### 3.6. `108Tasks`
- **ลักษณะ:** Tasks management (TypeScript / Node) (อ้างอิง `108Tasks#1`)
- **ไฟล์เป้าหมาย:** `.github/workflows/ci.yml`

```yaml
name: ci

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:
    name: "test · build"
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - name: Install dependencies
        run: npm ci
      - name: Run tests
        run: npm test
      - name: Build
        run: npm run build --if-present
```

---

### 3.7. `108-deploy` (Infra Shell)
- **ลักษณะ:** Scripts for deployment orchestration (Shell)
- **ไฟล์เป้าหมาย:** `.github/workflows/lint.yml`

```yaml
name: lint

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

concurrency:
  group: lint-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  shellcheck:
    name: "shellcheck"
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run shellcheck
        run: |
          find . -name "*.sh" -not -path "*/.*" -exec shellcheck -x {} +
```

---

### 3.8. `ting-service-chart` (Infra Helm)
- **ลักษณะ:** Core Helm chart used across the entire ecosystem
- **ไฟล์เป้าหมาย:** `.github/workflows/lint.yml`

```yaml
name: lint

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

concurrency:
  group: lint-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  helm-lint:
    name: "helm lint"
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: azure/setup-helm@v4
        with:
          version: v3.14.0
      - name: Lint Chart
        run: |
          helm lint .
```

---

## 4. Rollout Checklist

- [x] จัดทำมาตรฐาน CI Specification และ Workflow Templates ให้ครบทั้ง 8 repo
- [ ] เปิด Issue แยกราย repo เพื่อนำ workflow เหล่านี้ไปใส่ใน repo เป้าหมาย
- [ ] สำหรับ public repos ตรวจสอบว่าสามารถรันผ่าน GitHub Actions (`ubuntu-latest`) ได้ทันทีโดยไม่ติด queue ขององค์กร
