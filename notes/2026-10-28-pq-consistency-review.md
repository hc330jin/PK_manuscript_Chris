# P、Q 與 pricing kernel 一致性檢查

檢查日期：2026-10-28。程式來源為 `HWTeng-FIMA-Lab/2026-Chris-PK` 的 GitHub main，commit `45b06d2`。本機該 repo 有未提交變更且落後遠端，因此以獨立暫存 bare clone 讀取最新版本，沒有更動本機原 repo 或校準結果。

## 結論

目前沒有發現「用 vanilla option 錯誤取代 inverse option」或「PK 把 BTC risk-neutral density 當成 USD density」的問題。Q 實證校準用 USD numeraire，輸出的 option price 則是 BTC 報價；PK 仍使用同一個 Q^USD。P 估計的是 BTC/USD 的實際 log return 分布，本身沒有 inverse/vanilla payoff 的選擇。

本次檢查包含程式閱讀、輸出日期覆蓋檢查及獨立代數／數值檢查，沒有重跑 MATLAB 校準，也沒有驗證遠端桌機未上傳的檔案。

## Payoff 與單位轉換

設 omega = +1（call）或 -1（put）。BTC inverse option 的到期給付為

\[
H_T^{BTC}=\frac{[\omega(S_T-K)]^+}{S_T}.
\]

到期 USD 價值為 \(S_T H_T^{BTC}=[\omega(S_T-K)]^+\)。因此在 repo 維持的零利率設定下，

\[
C_t^{BTC}=\frac{E_t^{Q^{USD}}[S_T H_T^{BTC}]}{S_t}
=E_t^{Q^{USD}}\left[\left[\omega\left(\frac{S_T}{S_t}-\frac{K}{S_t}\right)\right]^+\right].
\]

這正是 `calc_SVCJ_QUSD_realdata_fast.m` 實作的 normalized vanilla payoff。分母是**當期** S_t；不能改成在 Q^USD 下直接平均 H_T^BTC。若改用 BTC numeraire，必須同時改變機率測度，不能只改 payoff。

簡單數值檢查：令 Q^USD 下 G=S_T/S_t 以各 1/2 的機率等於 0.8、1.2，且 K/S_t=1。Call 與 put 的正確 BTC 價格皆為 0.1。直接在同一 Q^USD 下平均 inverse payoff，卻會分別得到 0.083333 與 0.125。以 BTC numeraire 對應的權重 0.4、0.6 計算 inverse payoff，則都恢復為 0.1。

Deribit 官方文件也區分 BTC 到期給付與當期 USD/BTC 報價轉換：[Inverse Options](https://support.deribit.com/hc/en-us/articles/31424939096093-Inverse-Options)。本研究使用 2024 現金結算資料；該頁另述的 2026 結算流程調整不適用於研究樣本。

## 程式追蹤

以下連結固定至本次檢查的 commit：

- [P return loader](https://github.com/HWTeng-FIMA-Lab/2026-Chris-PK/blob/45b06d2/coding/P/load_BTC_P_returns.m)：使用 `diff(log(values))`，狀態是 BTC/USD log return。
- [Q panel loader](https://github.com/HWTeng-FIMA-Lab/2026-Chris-PK/blob/45b06d2/coding/Q/load_closing_trade_Q_panel.m#L8-L17)：S0 為選定交易的 contemporaneous index price；市場 premium 保留 BTC，USD premium = BTC premium × S0。
- [Q MC pricer](https://github.com/HWTeng-FIMA-Lab/2026-Chris-PK/blob/45b06d2/coding/Q/calc_SVCJ_QUSD_realdata_fast.m#L38-L61)：模擬 Q^USD 的 G=S_T/S0，再以 K/S0 計算 normalized vanilla payoff。第 72–85 行的排序／累積和實作等價於直接取 payoff 平均。
- [Q optimizer](https://github.com/HWTeng-FIMA-Lab/2026-Chris-PK/blob/45b06d2/coding/Q/calibrate_one_real_SVCJ_Q_day.m#L42)：實際使用 bounded trust-region-reflective `lsqnonlin`。
- [P predictive simulator](https://github.com/HWTeng-FIMA-Lab/2026-Chris-PK/blob/45b06d2/coding/PK/simulate_SVCJ_P_predictive_returns.m)：參數與視窗末端 latent variance 保持配對，累積七日 BTC/USD log returns。
- [Q predictive simulator](https://github.com/HWTeng-FIMA-Lab/2026-Chris-PK/blob/45b06d2/coding/PK/simulate_SVCJ_QUSD_returns.m#L14-L41)：與 Q pricer 使用相同的 USD jump compensator、Bernoulli jump 與 variance truncation。
- [PK driver](https://github.com/HWTeng-FIMA-Lab/2026-Chris-PK/blob/45b06d2/coding/PK/run_SVCJ_density_PK.m#L73-L88)：P、Q 使用相同 horizon，Q 來源指向全年校準表。
- [Density ratio](https://github.com/HWTeng-FIMA-Lab/2026-Chris-PK/blob/45b06d2/coding/PK/estimate_SVCJ_density_PK.m#L83-L95)：在共同 support 上計算 q/p，未額外將 Q 改成 BTC measure。

在這個最新版本中未找到 Q simulation recovery experiment 的獨立 MATLAB 產生程式，故不能聲稱已核驗方法章 recovery 表格的整個生成流程。以上結論適用於目前可讀取的 Q 實證 pricer、P pipeline 與 PK pipeline。

## 全年結果存在，收斂問題仍在

- P 的 `real_SVCJ_P_w1000_50000iter/daily_SVCJ_P_parameters.csv` 有 366 個不同日期，完整覆蓋 2024-01-01 至 2024-12-31；`convergence_pass` 為真的日期是 0/366。
- Q 的 `real_SVCJ_Q_window_15min_100000paths/daily_SVCJ_Q_parameters.csv` 同樣有 366 個不同日期，完整覆蓋全年；所有儲存的 optimizer exitflag 都大於零。這只能證明完成與停止狀態，不等於全域最優或參數已被識別。
- 正式研究與正文 EDA 區間為 2024 全年；2024 之前的 spot 資料用於 P 的歷史 rolling windows。第一個 1000-return window 為 2021-04-06 至 2023-12-31。
- 一月 PK 是方法示範與 P 校準改良期間的 pilot，不能因為已有全年參數檔，就把現有一月圖當成完成的全年 PK 結果。

## 另外發現的離散化問題，這次不修改

Q pricer 與 PK Q simulator 均用 `jump = U < lambda`，亦即每日 Bernoulli jump，但 log drift 使用連續時間 Poisson 補償 `-lambda*kappaJ`。對此離散一步，零利率下的條件期望是

\[
E[G_{k+1}/G_k\mid\mathcal F_k]
=e^{-\lambda\kappa_J}(1+\lambda\kappa_J),
\]

而非嚴格等於 1。代入全年儲存的 Q 參數後，最大一步偏差約為 `5.29646e-6`（2024-04-16）。這是解析計算的 drift 偏差，不是實測 option price error，也不是已量化的 PK 誤差。

若保留 Bernoulli 日模型，精確的一步補償應為 `-log(1+lambda*kappaJ)`；另一條路是讓跳躍模擬與 Poisson 模型一致。選定後需同步處理 Q 校準、Q recovery simulation、PK simulator 與既有結果，不能只改其中一處。本次只記錄，未變動程式或重新校準。

此外，P daily close 與 Q 08:00 UTC 的 conditioning clocks 仍未對齊；P 與 Q 的報酬狀態、幣別及 horizon 一致，不代表資訊時間完全同步。

## 2024 正文 EDA 的重算來源

使用同 commit 的 `data/btc_usd_daily_for_P.csv`；它與目前本機資料 SHA-256 一致：`839035c6eb0163697a4dfc813764a78cbd2b7e84991ca839ef547b3abaf19104`。

重現命令（在本 manuscript repo 中）：

```sh
python3 scripts/build_2024_spot_statistics.py /path/to/2026-Chris-PK/data/btc_usd_daily_for_P.csv
```

腳本輸出正文表格 `Sections/02_data_tables/btc_returns_2024.tex` 並列出統計量。366 個 returns 僅以 2024 日期篩選，計算 1 月 1 日報酬時使用 2023-12-31 價格；標準差採 n−1，skewness 與 Pearson kurtosis 採經驗中心動差。獨立以 pandas/NumPy 計算核對，結果一致。

Mean 0.002160、SD 0.026733、skewness 0.169711、kurtosis 3.995338、min -0.083555、median 0.001947、max 0.096377、JB 16.865049。JB 的漸近 p-value 約 0.00021767。
