#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
fidget_fourier.py
================================================================
イベントカメラの「イベント発生率(event rate)」をFourier解析して、
リズミカルな動き(= fidgeting様の振動)を検出するスクリプト。

手法のベース:
  Hamann et al. (2025) "Fourier-Based Action Recognition for Wildlife
  Behavior Quantification with Event Cameras", Advanced Intelligent Systems.
  (arXiv:2410.06698)
  → ペンギンの羽ばたき(約2Hz)を、イベント発生率の時系列に現れる
     正弦波的な信号として捉え、その周波数帯のエネルギーを見るだけで
     行動を識別する、という考え方。DNNに匹敵する性能を、5桁少ない
     パラメータ(=超軽量・エッジ向き)で達成している。

  本スクリプトはこの「イベント率 → Fourier帯域エネルギー → 閾値判定」
  という最小構成を、fidgeting検出向けにそのまま実装したもの。
  EEPPRのような精密な単一周波数推定ではなく、「リズミカルな動きが
  起きているか/その強さ」を見るのが目的(=適応的介入のトリガ判定に対応)。

読み込み:
  あなたのOpenEB環境に合わせて metavision_core の EventsIterator を使用。
  .raw / .hdf5 のどちらも読める。EventsIterator は delta_t マイクロ秒ごとに
  イベントを区切って返すので、その1区切り=時系列信号の1サンプルになる。
  この「区切りごとに処理する」構造が、そのままリアルタイム運用の形でもある。
  (※ Metavisionを使いたくない場合は load_rate_signal を expelliarmus 版に
     差し替えるだけでよい。)

使い方の例:
  source ~/naist_event/.venv/bin/activate
  python fidget_fourier.py -f ~/naist_event/EEPPR/dataset/06_spider/06_spider.raw \
         --roi 870 259 1096 485 --start_s 10 --dur_s 2 --band 2 6
================================================================
"""
import argparse
import numpy as np
import matplotlib
matplotlib.use("Agg")            # GUI不要(tkinter不要)。画像はPNGに保存する
import matplotlib.pyplot as plt


# ----------------------------------------------------------------
# 1) 読み込み:イベントを delta_t ごとに区切り、ROI内の極性別イベント数を数える
#    → 返り値は「イベント発生率の時系列」(正/負の2本)とサンプリング周波数 fs
# ----------------------------------------------------------------
def load_rate_signal(path, bin_us, roi=None):
    from metavision_core.event_io import EventsIterator

    # mode="delta_t": 各反復で「直近 bin_us マイクロ秒ぶん」のイベント配列を返す
    mv_it = EventsIterator(input_path=path, mode="delta_t", delta_t=bin_us)
    height, width = mv_it.get_size()
    print(f"[センサ解像度] {width} x {height}")

    rate_pos, rate_neg = [], []
    for evs in mv_it:                       # evs: 構造化配列 ('x','y','p','t')
        if roi is not None and evs.size:
            x0, y0, x1, y1 = roi
            e = evs[(evs["x"] >= x0) & (evs["x"] < x1) &
                    (evs["y"] >= y0) & (evs["y"] < y1)]
        else:
            e = evs
        # この区切りの中で、正極性(p=1)/負極性(p=0)のイベントを数える
        rate_pos.append(int((e["p"] > 0).sum()) if e.size else 0)
        rate_neg.append(int((e["p"] <= 0).sum()) if e.size else 0)

    fs = 1e6 / bin_us                       # サンプリング周波数 [Hz]
    return np.asarray(rate_pos, float), np.asarray(rate_neg, float), fs


# ----------------------------------------------------------------
# 2) Fourier帯域エネルギー比:論文の「判定用の特徴量」そのもの
#    対象周波数帯 [f_lo, f_hi] にどれだけパワーが集中しているかを
#    全体に対する比として返す。これが大きい=その帯域で振動している。
# ----------------------------------------------------------------
def band_energy_ratio(sig, fs, f_lo, f_hi):
    s = sig - sig.mean()                    # 直流成分(平均的なイベント数)を除去
    s = s * np.hanning(len(s))              # 窓関数:端の不連続による漏れを抑える
    freqs = np.fft.rfftfreq(len(s), d=1 / fs)
    power = np.abs(np.fft.rfft(s)) ** 2     # パワースペクトル

    band = (freqs >= f_lo) & (freqs <= f_hi)
    total = power[freqs > 0.3].sum() + 1e-9 # 0.3Hz以下(ゆっくりしたドリフト)は除外
    ratio = power[band].sum() / total
    f_dom = freqs[band][np.argmax(power[band])] if band.any() else float("nan")
    return ratio, f_dom, freqs, power


# ----------------------------------------------------------------
# 3) 自己相関による周期推定:Fourierの補助。EEPPRの「相関ピーク間隔」と
#    同じ発想で、信号が何秒ごとに自分自身と似るか(=周期)を見る。
# ----------------------------------------------------------------
def autocorr_freq(sig, fs, f_lo, f_hi):
    s = sig - sig.mean()
    ac = np.correlate(s, s, mode="full")[len(s) - 1:]
    ac = ac / ac[0]                         # lag0 で正規化
    lo, hi = int(fs / f_hi), int(fs / f_lo) # 探索する lag の範囲(周波数帯に対応)
    if hi <= lo or hi > len(ac):
        return float("nan"), ac
    lag = lo + int(np.argmax(ac[lo:hi]))    # 帯域内で最初に強く相関する lag
    return fs / lag, ac


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--file", "-f", required=True, help=".raw または .hdf5 のパス")
    ap.add_argument("--roi", "-r", nargs=4, type=int,
                    metavar=("X0", "Y0", "X1", "Y1"), default=None,
                    help="解析する矩形ROI(省略で全画面)")
    ap.add_argument("--bin_ms", type=float, default=5.0,
                    help="時間ビン幅[ms]。fs = 1000/bin_ms [Hz]")
    ap.add_argument("--band", nargs=2, type=float, metavar=("F_LO", "F_HI"),
                    default=[2.0, 6.0], help="対象周波数帯[Hz](fidget想定)")
    ap.add_argument("--start_s", type=float, default=0.0, help="解析開始[s]")
    ap.add_argument("--dur_s", type=float, default=0.0,
                    help="解析する長さ[s](0で末尾まで)")
    ap.add_argument("--plot", default="fidget_fourier.png", help="出力PNG")
    args = ap.parse_args()
    f_lo, f_hi = args.band

    # --- 信号を作る ---
    bin_us = int(args.bin_ms * 1000)
    rate_pos, rate_neg = None, None
    rate_pos, rate_neg, fs = load_rate_signal(args.file, bin_us, args.roi)
    sig_full = rate_pos + rate_neg          # 正負を合算した活動量(極性別に見たいなら分けてもよい)

    # --- 解析区間を切り出す(録画の頭が静かな場合などに) ---
    i0 = int(args.start_s * fs)
    i1 = int((args.start_s + args.dur_s) * fs) if args.dur_s > 0 else len(sig_full)
    sig = sig_full[i0:i1]
    print(f"[信号] 全長 {len(sig_full)} サンプル / fs={fs:.0f}Hz、"
          f"解析区間 {args.start_s:.1f}-{(i1/fs):.1f}s ({len(sig)} サンプル)")
    if len(sig) < 16:
        raise SystemExit("解析区間が短すぎます。--start_s / --dur_s を見直してください。")

    # --- 特徴量を計算 ---
    ratio, f_dom, freqs, power = band_energy_ratio(sig, fs, f_lo, f_hi)
    f_ac, ac = autocorr_freq(sig, fs, f_lo, f_hi)
    print(f"[結果] 帯域[{f_lo}-{f_hi}Hz] エネルギー比 = {ratio:.3f}   "
          f"(大きいほど『その帯域で振動している』)")
    print(f"[結果] FFTピーク周波数 = {f_dom:.2f} Hz   自己相関の周期 = {f_ac:.2f} Hz")
    print(f"[平均イベント率] {sig.mean():.1f} events/{args.bin_ms:.0f}ms  "
          f"(動きの強さの目安)")

    # --- 可視化 ---
    fig, ax = plt.subplots(3, 1, figsize=(9, 8))
    tax = (np.arange(len(sig)) / fs) + args.start_s
    ax[0].plot(tax, sig, lw=0.8)
    ax[0].set_title("Event-rate signal (ROI)"); ax[0].set_xlabel("time [s]")
    b = (freqs >= 0) & (freqs <= max(f_hi * 3, 15))
    ax[1].plot(freqs[b], power[b])
    ax[1].axvspan(f_lo, f_hi, color="orange", alpha=0.2)  # 対象帯域
    ax[1].set_title("Power spectrum (band shaded)"); ax[1].set_xlabel("Hz")
    lags = np.arange(len(ac)) / fs
    hi = int(fs / f_lo)
    ax[2].plot(lags[:hi], ac[:hi])
    ax[2].set_title("Autocorrelation"); ax[2].set_xlabel("lag [s]")
    fig.tight_layout(); fig.savefig(args.plot, dpi=110)
    print(f"[保存] {args.plot}")


if __name__ == "__main__":
    main()
