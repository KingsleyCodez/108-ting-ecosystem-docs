# Helm-Based Deploy Porting Guide — 108 Ting Ecosystem

**Status:** Implementation Guide & Migration Roadmap (Resolves #14)  
**Standard Reference:** §4 of `VERSIONING_STANDARD.md` (*"Deploy through helm, never around it"*)  
**Reference Implementation:** `pos108-admin#541` → `pos108-admin#543`

---

## 1. Problem Statement & Baseline Survey

การสำรวจคลัสเตอร์เมื่อ 2026-09-01 พบว่า **15 จาก 38 releases มีสถานะ drift ออกจากค่า Helm values** เนื่องจากในอดีตมีการใช้ `kubectl set image` ทำให้ค่า image tag ที่เก็บอยู่ใน Helm values ล้าสมัยและกลายเป็นกับดัก:

| Release | Namespace | Stored Helm Tag (Date) | Live Pod Image (Date) | ผลกระทบหากสั่ง `helm upgrade` ปกติ |
|---|---|---|---|---|
| **tenant-orders** | tenants | `sha-2092b1c` (07-31) | `sha-04b29fa` (08-24) | **ย้อนหลัง ~1 เดือน (ร้านจริงพัง)** |
| **pos-sell** | prod | `sha-cdc7a95` (08-12) | `sha-b06f648` (09-01) | ย้อนหลัง ~3 สัปดาห์ (108plaza.com) |
| **pos-store** | prod | `sha-73f6f25` (08-12) | `sha-22379d7` (08-31) | ย้อนหลัง ~3 สัปดาห์ (shop.108plaza.com) |
| **control-plane**| prod | `sha-b9508ac` (08-14) | `sha-972265a` (09-01) | ย้อนหลัง ~2.5 สัปดาห์ |
| **pos108** | staging | `sha-c26fcf5` (08-14) | `sha-972265a` (09-01) | ย้อนหลัง ~2.5 สัปดาห์ |
| **pos-orders** | staging | `sha-47d1258` (08-12) | `sha-04b29fa` (08-24) | ย้อนหลัง ~3 สัปดาห์ |
| **t-365** | tenants | `sha-9075172` (08-31) | `sha-972265a` (09-01) | ย้อนหลัง 1 วัน |
| **chat** | prod & staging | `sha-1a3aa6c` | `sha-d3d4708` | drift |
| **customer/loyalty/secrets** | staging | `sha-bc26337` / `sha-419e0d9` | `sha-f850a35` | drift |
| **identity / identity-jobs** | staging | `sha-7bdc9d1` | `sha-c3e3bf4` | drift |
| **notify** | staging | `sha-5eed3c8` | `sha-e211fb0` | drift |
| **heros-web** | staging | `sha-a1bb7ae` | `sha-b51f1b9` | drift |

---

## 2. The Golden Deploy Pattern (จาก `pos108-admin#543`)

ทุก Workflow ของการ deploy ต้องรันการอัปเกรดผ่าน Helm และส่ง parameters ครบทั้ง `image.repository` และ `image.tag` เสมอ เพื่อให้การ deploy ทุกครั้งปิดช่องว่างของ drift โดยอัตโนมัติ:

```bash
# 1. รันการอัปเกรดผ่าน Helm พร้อมส่ง repository และ tag ชัดเจน
helm upgrade <rel> "$CHART_PATH" \
  -n <ns> \
  --reset-then-reuse-values \
  --set image.repository="ghcr.io/108-plaza/<repo>" \
  --set image.tag="sha-${SHORT_SHA}" \
  --wait \
  --timeout 3m

# 2. ตรวจสอบสถานะ Rollout ของ Deployment
kubectl -n <ns> rollout status deploy/<rel> --timeout=180s

# 3. ด่านสำคัญ: ตรวจสอบว่ามี Replica ที่พร้อมรันจริง (ป้องกันเคส scaled to 0)
READY=$(kubectl -n <ns> get deploy/<rel> -o jsonpath='{.status.readyReplicas}')
if [ -z "$READY" ] || [ "$READY" -lt 1 ]; then
  echo "CRITICAL: Deploy finished but readyReplicas < 1 (found: '$READY')"
  exit 1
fi
echo "Verified: deploy/<rel> is serving with $READY ready replica(s)"
```

---

## 3. Two Critical Traps & Prevention

### Trap 1: อันตรายจากการใช้ `--reuse-values` ธรรมดา
- **สาเหตุ:** `--reuse-values` จะรักษาเฉพาะค่าเดิมที่เคยบันทึกไว้ใน release แต่หากตัว chart มีการเพิ่มคีย์ใหม่ในภายหลัง ค่าใหม่จะกลายเป็น `nil` นำไปสู่ข้อผิดพลาด `nil pointer evaluating interface {}.type`
- **วิธีแก้:** ต้องใช้ **`--reset-then-reuse-values`** (รองรับตั้งแต่ Helm 3.14+ ขึ้นไป โดยคลัสเตอร์ของ 108 Ting ทำงานบน Helm 3.21)

### Trap 2: การตรวจสอบ Diff ต้องเทียบกับ Live Object เท่านั้น
- **ข้อผิดพลาด:** การเปรียบเทียบ `helm get manifest` กับ `helm upgrade --dry-run` ไม่สามารถตรวจจับการแก้ manual ด้วย `kubectl patch` ได้
- **ผลเสียที่เคยเกิดขึ้น:** release `control-plane` เมื่อ 08-14 ถูก revert ค่า probe `/health/ready` กลับเป็น `/health` อย่างเงียบ ๆ ทำให้ pod รายงานว่า Ready ทั้งที่ฐานข้อมูลล่ม
- **วิธีแก้:** ให้ diff dry-run กับ live Kubernetes object:
  ```bash
  helm upgrade <rel> "$CHART_PATH" -n <ns> --reset-then-reuse-values --set ... --dry-run | \
    kubectl diff -f -
  ```
  หรือดึง `kubectl get deploy <rel> -o yaml` มาตรวจสอบ diff ก่อนการ rollout

---

## 4. Priority Rollout Roadmap

การ port workflow ให้แบ่งทำทีละ repo ผ่าน PR เดี่ยว โดยเรียงตามระดับความเสี่ยงของผลกระทบ (Prod และร้านจริงก่อน Staging):

### Phase 1: High Blast Radius / Production & Real Shops
1. **`pos108-orders`**
   - **Releases:** `tenant-orders`, `pos-orders`
   - **ความเสี่ยง:** รุนแรงที่สุด — `tenant-orders` drift ถึง 1 เดือนบนร้านค้าจริง
2. **`pos108-core`**
   - **Releases:** `control-plane`, `pos108`, และทุก tenant `t-*` (เช่น `t-365`)
   - **ความเสี่ยง:** แกนหลักของทั้งระบบ POS และ API
3. **`pos108-sell`**
   - **Releases:** `pos-sell`
   - **ความเสี่ยง:** หน้าร้านหลัก `108plaza.com`
4. **`pos108-store`**
   - **Releases:** `pos-store`
   - **ความเสี่ยง:** หน้าร้านออนไลน์ `shop.108plaza.com`

### Phase 2: Central Platform Services (Staging)
5. **`108-platform-services`**
   - **Releases:** `customer`, `loyalty`, `notify`, `secrets`, `identity`, `identity-jobs`
   - **ความเสี่ยง:** Path-filtered 5 services
6. **`108heros-web`**
   - **Releases:** `heros-web` (staging)
7. **`Livechat-Platform`**
   - **Releases:** `chat` (prod และ staging)
   - **หมายเหตุ:** ต้องตรวจสอบขั้นตอนการ deploy ปัจจุบันก่อนนำ workflow มาปรับใช้

---

## 5. Deployment Step Template ใน GitHub Actions

ตัวอย่าง step สำหรับใส่ใน `.github/workflows/deploy.yml` หรือ `release-image.yml`:

```yaml
      - name: Deploy to K3s via Helm
        env:
          KUBECONFIG_DATA: ${{ secrets.KUBECONFIG }}
          CHART_PATH: "deploy/helm/ting-service"
          RELEASE_NAME: "pos-sell"
          NAMESPACE: "prod"
          IMAGE_REPO: "ghcr.io/108-plaza/pos108-sell"
          IMAGE_TAG: "sha-${{ steps.meta.outputs.short_sha }}"
        run: |
          mkdir -p ~/.kube
          echo "$KUBECONFIG_DATA" | base64 -d > ~/.kube/config
          chmod 600 ~/.kube/config

          echo "Upgrading Helm release ${RELEASE_NAME} in namespace ${NAMESPACE}..."
          helm upgrade "${RELEASE_NAME}" "${CHART_PATH}" \
            -n "${NAMESPACE}" \
            --reset-then-reuse-values \
            --set image.repository="${IMAGE_REPO}" \
            --set image.tag="${IMAGE_TAG}" \
            --wait \
            --timeout 3m

          echo "Verifying rollout status..."
          kubectl -n "${NAMESPACE}" rollout status deploy/"${RELEASE_NAME}" --timeout=180s

          echo "Verifying active ready replicas..."
          READY=$(kubectl -n "${NAMESPACE}" get deploy/"${RELEASE_NAME}" -o jsonpath='{.status.readyReplicas}')
          if [ -z "$READY" ] || [ "$READY" -lt 1 ]; then
            echo "ERROR: ${RELEASE_NAME} has no ready replicas!"
            exit 1
          fi
          echo "Success: ${RELEASE_NAME} is serving with ${READY} replica(s)."
```
