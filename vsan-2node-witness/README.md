# vSAN 2-node + Witness 検証ログ（ESXi 8.x）

本ドキュメントは、**ESXi 8.x 環境における vSAN 2ノード構成 + Witness** を  
**Witness Appliance を使わず、物理 ESXi ホストを Witness として構成**した検証記録である。

Broadcom 移行後の vSAN / vCenter 8.x 環境で実際に発生した **詰まりポイント・回避策・障害試験結果** を含む。

---

## 構成概要

### 物理構成

NodeA (ESXi 8.0u3) ---- 10GbE ---- NodeB (ESXi 8.0u3)
| |
+---------- Mgmt (1GbE) -----------+
|
Witness (ESXi 8.0u3)


- NodeA / NodeB：データノード
- Witness：投票・クォーラム専用（データ保持なし）
- vMotion：10GbE
- vSAN：10GbE
- Management：1GbE

---

## 前提条件（重要）

- vSAN 2ノード構成では **Witness が必須**
- Witness は **クラスタに参加させない**
- Witness には **Flash / SSD デバイスが1本必須（キャッシュ、キャパシティそれぞれ）**
- RAID 論理ディスクは **Witness用途に不可**
- VM Home（名前空間）と VMDK は **別オブジェクト**
- デフォルトストレージポリシー（FTT=1）のままでは VM 作成不可

## 注意事項：Witness Appliance の入手について

本検証では **Witness Appliance（OVA）を使用していない**。

### 理由

- 現在（Broadcom 移行後）、**vSAN Witness Appliance は個人ユーザーでは入手が非常に困難**である
- Broadcom Support Portal では以下の制約がある：
  - 企業アカウント（Site ID / 契約）が必要
  - 個人アカウントではダウンロード権限が付与されない場合が多い
- 旧 VMware Customer Connect 時代の公開リンクは多くが無効化されている

### そのための代替手段

本検証では、以下の構成を採用した：

- **物理 ESXi ホストを Witness として使用**
- Witness はクラスタに参加させず、vSAN 構成時に監視ホストとして指定
- Flash / SSD デバイスを 1 本以上用意することで、Witness の要件を満たす

### 補足

本手法は **公式に非推奨ではないが、一般的な構成例として紹介されることは少ない**。  
しかし、検証用途や自己学習環境では **十分に実用的な代替手段**である。

---

## 1. ESXi ホスト準備

### NodeA / NodeB
- ESXi 8.0u3 インストール
- ローカルディスクは vSAN 用に使用
- Management vmk 作成

### Witness
- ESXi 8.0u3 インストール
- **HBA モードで Flash/SSD を1本直結**
- Datastore 作成不要
- Management vmk 作成

---

## 2. ネットワーク設計

### VMkernel NIC
| 用途 | vmk | 備考 |
|----|----|----|
| Management | vmk0 | 1GbE |
| vSAN | vmk1 | 10GbE |
| vMotion | vmk2 | 10GbE |

- MTU：1500（全系統統一）
- vSAN / vMotion は同一NIC可（検証用途）

---

## 3. vCenter / クラスタ作成

1. vCenter Server 8.x デプロイ
2. Datacenter 作成
3. Cluster 作成
4. NodeA / NodeB をクラスタに追加
5. DRS / HA は任意（検証では OFF でも可）

---

## 4. vSAN 有効化

- Cluster → 構成 → vSAN → 有効化
- 構成タイプ：**シングルサイト**
- vSAN ESA：使用しない（OSA）

---

## 5. Witness 登録

- Cluster → 構成 → vSAN → Witness
- **Witness ESXi ホストを指定**
- Witness は **クラスタに追加しない**

### Witness vSAN vmk 登録（UIで出ない場合）
```bash
esxcli vsan network ipv4 add -i vmk1
esxcli vsan network list
```

## 6. フォルトドメイン設定（重要）

データノードのみ手動設定する

FD-NodeA → NodeA
FD-NodeB → NodeB

Witness は 自動的に別ドメイン扱いされる（UI非表示）。
## 7. vSAN Datastore 作成
    ディスクグループ作成
    Datastore 作成確認

## 8. VM作成時の地雷と回避策（FTT=0）
問題
    デフォルトポリシー（FTT=1）では
    障害ドメイン不足で VM 作成不可

回避策：FTT=0 ポリシー作成
    ストレージポリシー → 新規作成
    vSAN ストレージルールを有効化
    許容される障害の数：0
    ポリシー名例：vSAN-FTT0

VM作成時の注意
    VM全体（VM Home）に vSAN-FTT0 を指定
    仮想ディスクも全て vSAN-FTT0

※ ディスクだけ指定しても失敗する（VM Homeが原因）

## 9. 性能検証（CrystalDiskMark）
観測結果（FTT=0）
    Read：配置ノード一致時は高速
    Read：vMotion後は大幅低下
    Write：一貫して遅い（仕様）

学び
    vMotionはデータを動かさない
    FTT=0 は 配置依存性能

## 10. 障害試験
ケース1：データを持たないノード停止
    VM：生存
    性能：低下
    vSAN Health：赤

ケース2：データ保持ノード停止（FTT=0）
    VM：生存するが I/O 詰まり
    OS挙動：不安定
    ノード復帰で回復

FTT=0 は「止まらないことがある」が「安全ではない」
## 11. 撤収手順
    vSAN Datastore 上の VM 削除
    Cluster → vSAN 無効化
    フォルトドメイン削除
    Witness 解除
    Witness ESXi シャットダウン
    vSAN / vMotion / vmk / Portgroup 削除

まとめ
    vSAN は「性能装置」ではなく「可用性装置」
    FTT=0 は検証用途限定
    VM Home と VMDK の違いが最大の地雷
    UIだけでなく オブジェクト配置の理解が必須
