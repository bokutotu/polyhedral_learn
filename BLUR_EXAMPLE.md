## 1. 元コードとステートメントタグ

```c
// S1 : 横方向 3 点平均  (A → tmp)
for (int i = 0;   i < N;   ++i)
  for (int j = 1; j < M-1; ++j)
    tmp[i][j] = (A[i][j-1] + A[i][j] + A[i][j+1]) / 3;

// S2 : 縦方向 3 点平均  (tmp → B)
for (int i = 1;   i < N-1; ++i)
  for (int j = 1; j < M-1; ++j)
    B[i][j] = (tmp[i-1][j] + tmp[i][j] + tmp[i+1][j]) / 3;
```

*タグ変数*  
\[
\sigma =
\begin{cases}
0 & (\text{S₁})\\
1 & (\text{S₂})
\end{cases}
\]

---

## 2. パラメータ空間と反復ドメイン
$$
P = { (N,M)\in\mathbb Z_{>0}^2 }
$$

\[
\begin{aligned}
D_{S1} &= \{\, (i,j,0) \mid 0\le i<N,\; 1\le j<M-1 \,\},\\
D_{S2 ` | “1 行上へ” |

形式的には  
\[
R_{\delta_i}= \bigl\{\,((i,j,0),(i+\delta_i,j,1))\bigr\}.
\]

---

## 5. 融合可否のチェック

### 5.1 素朴融合候補  
$$
\Theta_{naÏve}(i,j,\sigma)= (i, j, \sigma)
$$

| 依存 | \(\delta = \Theta(\text{sink})-\Theta(\text{source})\) | 判定 |
|------|--------------------------------------|------|
| \(R_{\text{center}}\) | \((0,0,1)\)  | ✔ |
| \(R_{\text{up}}\)     | \((+1,0,1)\) | ✔ |
| \(R_{\text{down}}\)   | \((\color{red}{-1},0,1)\) | **✘** (外側 \(i\) 次元が負) |

→ **完全融合は不可能**。

---

### 5.2 パイプライン・スキューで合法化  
“縦ブラーを 1 行遅らせる”

\[
\boxed{\;\Theta_{\text{skew}}(i,j,\sigma)=\bigl(i+\sigma,\;j,\;\sigma\bigr)\;}
\]

*計算*  
\[
\delta = \bigl((i+\delta_i)+1,\;j,\;1\bigr) - \bigl(i,\;j,\;0\bigr)
       = \bigl(\delta_i+1,\;0,\;1\bigr)
\]

| 依存 | \(\delta\) under \(\Theta_{\text{skew}}\) | 判定 |
|------|-------------------------------------------|------|
| \(R_{\text{center}}\) | \((1,0,1)\) | ✔ |
| \(R_{\text{up}}\)     | \((2,0,1)\) | ✔ |
| \(R_{\text{down}}\)   | \((0,0,1)\) | ✔（最外 \(i\)=0 でも最内 \(\sigma=1>0\)） |

→ **3 依存すべてが辞書式で正**。  
したがって **スキュー付き融合は合法**。

> **実行イメージ**  
> ```
> 行 0: S1
> 行 1: S1 → S2
> 行 2: S1 → S2
> ...
> 行 N-2: S1 → S2
> 行 N-1: S2
> ```

---

## 6. ISL での制約モデル（要旨）

```c
// 1. 反復集合 (σ は暗黙に 0/1)
D = "{ S1[i,j] : 0<=i<N  and 1<=j<M-1; \
       S2[i,j] : 1<=i<N-1 and 1<=j<M-1 }";

// 2. 書き/読み
writes = "{ S1[i,j] -> tmp[i,j]; S2[i,j] -> B[i,j] }";
reads  = "{ S1[i,j] -> A[i,j-1]; S1[i,j] -> A[i,j];   \
            S1[i,j] -> A[i,j+1];                       \
            S2[i,j] -> tmp[i-1,j]; S2[i,j] -> tmp[i,j];\
            S2[i,j] -> tmp[i+1,j] }";

// 3. データフロー
isl_union_map *flow =
    isl_union_map_compute_flow(reads, writes,
                               /*schedule=*/NULL, NULL, NULL);

// 4. スキュー付きスケジュール
theta = "{ S1[i,j] -> [i    , j, 0]; \
           S2[i,j] -> [i + 1, j, 1] }";

// 5. 合法性判定
isl_bool ok = isl_union_map_lex_lt_union_map(
                 flow,
                 isl_union_map_from_multi_aff(theta));
```

`ok` が `isl_bool_true` なら融合スケジュールは依存を守っている。

---

## 7. まとめ

* **依存多面体は距離 ±1 まで網羅**しなければ可否を誤る。  
* 素朴融合は `R_down` で失敗。  
* **スケジューリング関数**  
  \[
  \Theta(i,j,\sigma)=\bigl(i+\sigma,\;j,\;\sigma\bigr)
  \]
  で **1 行パイプライン化**すれば合法融合が成立。  
* ISL／Polly は「依存制約 + 線形スケジュール」を ILP に落として  
  同様の判断とスキュー量探索を自動化してくれる。

これが完全版の BLUR カーネル依存解析 & フュージョン資料です。  
追加でタイル化や実コードのベンチ結果まで見たい場合は遠慮なくどうぞ！&= \{\, (i,j,1) \mid 1\le i<N-1,\; 1\le j<M-1 \,\}.
\end{aligned}
\]

---

## 3. アクセス写像

| Stmt | 配列 | R/W | 写像 \(f(i,j)\) |
|------|------|-----|-----------------|
| S₁   | `A`   | R | \((i,j-1),\,(i,j),\,(i,j+1)\) |
| S₁   | `tmp` | **W** | \((i,j)\) |
| S₂   | `tmp` | **R** | \((i-1,j),\,(i,j),\,(i+1,j)\) |
| S₂   | `B`   | **W** | \((i,j)\) |

---

## 4. RAW 依存多面体（3 種類、すべて `tmp` に関係）

| 名称 | 行オフセット \(\delta_i\) | 多面体（ISL 風） | 距離の口語名 |
|------|--------------------------|------------------|--------------|
| \(R_{\text{center}}\) | 0  | `{ [i,j] -> [i,j]   : 1<=i<N-1 and 1<=j<M-1 }` | “同じ行” |
| \(R_{\text{up}}\)     | +1 | `{ [i,j] -> [i+1,j] : 0<=i<N-2 and 1<=j<M-1 }` | “1 行下へ” |
| \(R_{\text{down}}\)   | −1 | `{ [i,j] -> [i-1,j] : 1<=i<N-1 and 1<=j<M-1 }}
