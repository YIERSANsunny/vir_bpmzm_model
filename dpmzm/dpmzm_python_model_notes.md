# DPMZM Python 模型公式说明

本文档记录当前 `dpmzm/model.py` 中实际使用的 DPMZM 输出模型，便于和 VPI / MATLAB 仿真逐项核对。

---

## 1. 电压到相位的映射

当前模型采用等效差分驱动电压：

$$
\phi_I(t)=\frac{\pi V_I(t)}{V_{\pi,I}},
\qquad
\phi_Q(t)=\frac{\pi V_Q(t)}{V_{\pi,Q}},
\qquad
\phi_P(t)=\frac{\pi V_P(t)}{V_{\pi,P}}.
$$

其中三路电压为：

$$
V_I(t)=V_{DCI}+V_{RFI}(t)+V_{\mathrm{dither},I}(t),
$$

$$
V_Q(t)=V_{DCQ}+V_{RFQ}(t)+V_{\mathrm{dither},Q}(t),
$$

$$
V_P(t)=V_{DCP}+V_{\mathrm{dither},P}(t).
$$

若使用内置 RF 正弦源：

$$
V_{RFI}(t)=A_I\sin(2\pi f_{RF}t+\theta_I),
$$

$$
V_{RFQ}(t)=A_Q\sin(2\pi f_{RF}t+\theta_Q).
$$

注意：`V_RFI_amp` 和 `V_RFQ_amp` 是正弦**峰值幅度**，不是峰峰值。

---

## 2. I/Q 子 MZM 模型

理想子 MZM 的输出场包络为：

$$
H_I^{ideal}(t)=\cos\left(\frac{\phi_I(t)}{2}\right),
$$

$$
H_Q^{ideal}(t)=\cos\left(\frac{\phi_Q(t)}{2}\right).
$$

考虑有限消光比后，当前模型使用文档推导中的形式：

$$
H_I(t)=
\cos\left(\frac{\phi_I(t)}{2}\right)
+
\delta_I \exp\left(j\frac{\phi_I(t)}{2}\right),
$$

$$
H_Q(t)=
\cos\left(\frac{\phi_Q(t)}{2}\right)
+
\delta_Q \exp\left(j\frac{\phi_Q(t)}{2}\right).
$$

消光比到 `delta` 的换算为：

$$
\delta_k=
\frac{1}{\sqrt{ER_k}-1}
=
\frac{1}{10^{ER_{k,dB}/20}-1},
\qquad k\in\{I,Q\}.
$$

当：

$$
ER_{I,dB}=ER_{Q,dB}=30\ \mathrm{dB},
$$

有：

$$
\delta_I=\delta_Q\approx 0.032655.
$$

---

## 3. 父级 / 主调制器合成模型

当前 Python 模型把父级 DPMZM 合成写为论文形式：

$$
E_{out}(t)
=
\frac{E_{in}\sqrt{L_{common}}}{2}
\left[
(1+\delta_P)\sqrt{L_I}\,H_I(t)
+
\gamma_P\sqrt{L_Q}\,H_Q(t)\exp(j\phi_P(t))
\right].
$$

其中：

$$
L_{common}=L_{global}L_P,
$$

`L_global` 是全局插入损耗因子，`L_P` 是父级额外损耗因子，`L_I` / `L_Q` 是 I/Q 支路额外损耗因子。

代码对应为：

```python
field_sum = (
    (1.0 + delta_P) * amp_I * H_I
    + gamma_P * amp_Q * H_Q * np.exp(1j * phi_P)
)

E_out = E_in * amp_common * 0.5 * field_sum
```

在默认模型中：

$$
\gamma_P=1.
$$

`gamma_P` 仅作为兼容/实验参数保留，不再由 `ER_P_dB` 自动生成。

### 3.1 父级有限消光比

父级 / 主调制器消光比当前用论文中的 `delta_P E_{I,out}` 项表示：

$$
\delta_P=
\frac{1}{\sqrt{ER_P}-1}
=
\frac{1}{10^{ER_{P,dB}/20}-1}.
$$

当：

$$
ER_{P,dB}=30\ \mathrm{dB},
$$

有：

$$
\delta_P\approx 0.032655.
$$

因此默认父级合成等价于：

$$
E_{out}(t)
=
(1+\delta_P)E_{I,out}(t)
+
E_{Q,out}(t)\exp(j\phi_P(t)).
$$

---

## 4. 理想情况下的归一化传输函数

当：

$$
\delta_I=\delta_Q=\delta_P=0,\qquad
\gamma_P=1,\qquad
L_{common}=L_I=L_Q=1,
$$

有：

$$
E_{out}(t)=
\frac{E_{in}}{2}
\left[
\cos\left(\frac{\phi_I(t)}{2}\right)
+
\cos\left(\frac{\phi_Q(t)}{2}\right)
\exp(j\phi_P(t))
\right].
$$

归一化光功率传输为：

$$
T(t)
=
\frac{|E_{out}(t)|^2}{|E_{in}|^2}
$$

$$
=
\frac{1}{4}
\left[
A^2+B^2+2AB\cos(\phi_P)
\right],
$$

其中：

$$
A=\cos\left(\frac{\phi_I}{2}\right),
\qquad
B=\cos\left(\frac{\phi_Q}{2}\right).
$$

展开后：

$$
T(t)
=
\frac{1}{8}
\left[
2
+
\cos(\phi_I)
+
\cos(\phi_Q)
+
4
\cos\left(\frac{\phi_I}{2}\right)
\cos\left(\frac{\phi_Q}{2}\right)
\cos(\phi_P)
\right].
$$

注意最后一项必须是半角项：

$$
4
\cos\left(\frac{\phi_I}{2}\right)
\cos\left(\frac{\phi_Q}{2}\right)
\cos(\phi_P),
$$

不是：

$$
4\cos(\phi_I)\cos(\phi_Q)\cos(\phi_P).
$$

---

## 5. 光功率、PD 电流和电谱口径

输出光功率为：

$$
P_{opt}(t)=|E_{out}(t)|^2.
$$

进入 PD 的光功率为：

$$
P_{PD}(t)=\mathrm{pd\_tap}\cdot P_{opt}(t).
$$

PD 电流为：

$$
I_{PD}(t)=R\cdot P_{PD}(t),
$$

其中 `R = Responsivity`，默认：

$$
R=0.786\ \mathrm{A/W}.
$$

电谱中，若：

```python
vpi_compatible_dbm = True
```

则按 VPI 常见的 1 Ω 显示口径：

$$
P_{elec}\propto I^2.
$$

若：

```python
vpi_compatible_dbm = False
```

则按物理负载：

$$
P_{elec}\propto I^2R_{load}.
$$

---

## 6. 当前 VPI demo 参数映射

`dpmzm/vpi_demo.py` 当前默认匹配 VPI 全局设置：

$$
BitRateDefault=10\ \mathrm{GHz},
$$

$$
SampleRateDefault=16\times BitRateDefault=160\ \mathrm{GSa/s},
$$

$$
TimeWindow=\frac{65536}{BitRateDefault}=6.5536\ \mu s.
$$

因此：

$$
N=1048576,
\qquad
RBW=\frac{1}{TimeWindow}=152587.890625\ \mathrm{Hz}.
$$

截图中的 VPI 电压源按 `LowerArmPhaseSense` 分别映射。

MZM 标准输出可写为：

$$
E_{out}
=
E_{in}
\cos\left[
\frac{\pi(V_{upper}-V_{lower})}{2V_\pi}
\right]
\exp\left[
j\frac{\pi(V_{upper}+V_{lower})}{2V_\pi}
\right].
$$

若 `LowerArmPhaseSense = NEGATIVE`，同一电压使上下臂相位反向：

$$
V_{upper}=V,\qquad V_{lower}=-V.
$$

因此：

$$
E_{out}=E_{in}\cos\left(\frac{\pi V}{V_\pi}\right),
$$

公共指数相位抵消。这适用于 I/Q 子调制器。

若 `LowerArmPhaseSense = POSITIVE`，同一电压使上下臂相位同向：

$$
V_{upper}=V,\qquad V_{lower}=V.
$$

因此：

$$
E_{out}=E_{in}\exp\left(j\frac{\pi V}{V_\pi}\right),
$$

cos 幅度项为 1，仅留下公共相位。这适用于当前 VPI 截图里的父级 / 主调制器。

因此 demo 中使用：

```python
child_drive_gain = 2.0
parent_drive_gain = 1.0
rf_drive_gain = 2.0
```

映射到 Python 模型的等效电压：

| VPI 参数 | Python 等效参数 |
|---|---|
| I 子 MZM DC = 2.5 V | `V_DCI = 5.0 V` |
| Q 子 MZM DC = 2.5 V | `V_DCQ = 5.0 V` |
| P 父级 DC = 1.25 V | `V_DCP = 1.25 V` |
| RF peak = 1.0 V | `V_RFI_amp = V_RFQ_amp = 2.0 V` |

此时：

$$
\phi_I=\phi_Q=\pi,
\qquad
\phi_P=\frac{\pi}{4},
$$

对应 I/Q 子 MZM 最小传输点，以及父级 POSITIVE 模式下的公共相位移。

---

## 7. 当前和 VPI marker 的差异

用户给出的 VPI PD 电谱 marker 为：

| 分量 | VPI |
|---|---:|
| DC | -49 dBm |
| 1f | -74 dBm |
| 2f | -53 dBm |

当前 Python 默认 `rf_phase_Q = 90°`、父级 ER 使用论文 `delta_P` 形式时：

| 分量 | Python | VPI | 差值 |
|---|---:|---:|---:|
| DC | -45.00 dBm | -49.00 dBm | +4.00 dB |
| 1f | -65.01 dBm | -74.00 dBm | +8.99 dB |
| 2f | -72.35 dBm | -53.00 dBm | -19.35 dB |

这个差异说明：

1. VPI 的时间窗 / RBW 已经对齐后，离散 DC/1f/2f marker 仍没有对齐；
2. 差异主要不是采样率或 RBW 导致的；
3. 更可疑的是 VPI 中 `PhaseShift`、`LowerArmPhaseSense`、DiffMZ 端口符号和当前 Python 相位定义之间的等效关系。

诊断扫描发现，若使用：

```python
rf_phase_Q = 40 deg
pd_tap = 0.69
```

则三点更接近 VPI：

| 分量 | Python | VPI | 差值 |
|---|---:|---:|---:|
| DC | -48.23 dBm | -49.00 dBm | +0.78 dB |
| 1f | -73.85 dBm | -74.00 dBm | +0.15 dB |
| 2f | -53.89 dBm | -53.00 dBm | -0.89 dB |

这只是诊断组合，不代表最终物理参数一定应取这些值。

---

## 8. 当前重点待核对项

1. VPI `DiffMZ_DSM` 的 RF 端口输入电压是单臂电压、差分电压，还是内部再做 push-pull 映射；
2. `LowerArmPhaseSense = NEGATIVE / POSITIVE` 对输出场相位的等效符号；
3. 外部 `PhaseShift = 90 deg` 进入 Q 路后，在当前公式中应对应 `+90°`、`-90°`，还是叠加其它符号；
4. VPI `Power N` 模块是否引入额外光功率比例；
5. SignalAnalyzer 对 DC/1f/2f marker 的功率定义是 peak、RMS、单边谱还是双边谱。
