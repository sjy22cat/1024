# 复制下面整段给新思服务器上的 Codex

```text
请在当前解压目录实际执行 Attention 的功能验证、SMIC 28nm 标准单元综合、STA 和活动功耗分析；不要只输出计划或算子预检查。

先读取 README.md、synopsys/MEASUREMENT_CONTRACT.md 和所有脚本，执行 python3 verify_upload.py 检查 SHA256。主对象为 attention_head_16ctx_4dim_eval，默认 HEAD_DIM=128、TILE_SIZE=32、CONTEXT_PARALLEL=16、KV_LOAD_PARALLEL=16、MAX_CONTEXT=1152、SCALE_BITS=0x3db504f3。一次命令是一条 Q 对 N 条 K/V 计算一条 128 维 O。主报告 N=1152；同一硬件也测 N=128、512、513、1024 的运行时周期/能量。结果不能标作 GQA group、16-head 系统或物理 MXFP4 阵列。

目标是给出 Attention TOTAL、QK/PV MAC、TOTAL−MAC residual 三类：
1) 实测周期与延迟 ns；2) 映射后标准单元面积 µm²；3) 每完整命令的动态、漏电、总能量 pJ。需要可复核原始报告及机器可读数据，供返回本地后进行 MAC 扣除。不要更新模拟器或既有 calibration。

一、环境与约束
探测已有 dc_shell / vcs / pt_shell、DesignWare 授权模型及 SMIC 库。选择 SCC28NHKCP_HSC35P140_RVT_V0p2，RVT/SS/0.9V/125C，basic NLDM/NLPM；若没有该角，列明实际可用库并保留缺项，不能冒充同角结果。
复制 synopsys/site.example.json 为 synopsys/site.json，填真实 .db 和官方 DW sim_ver 路径。从工具 report_units 和库资料确认时间、电容、功率、面积单位，填写单位换算；5 fF 应转换为 0.005 pF，不得猜测工具单位。
在第一次 run_flow 运行前一并填写 read_vcd_time_unit_ns：依据本机 read_vcd -time 的 man 和实际 VCD timescale 确认该数值，不可默认它等于库时间单位。该字段未确认时，脚本会拒绝功耗分析。
以 1.000 ns 理想时钟先评估。setup/hold uncertainty=50/20ps，clock/input slew=50ps，数据输入 max/min=100/0ps（排除 clk/rst_n），数据输出 max/min=100/0ps，每输出负载5fF，max transition=200ps，仅 rst_n false path，外部同步解除复位。禁止额外 FSM 假路径、multicycle 或修改输入最小延迟掩盖违例。布局前，无 CTS/SPEF。
检查本版 Synopsys help/man，尤其 report_power 层次归属、get_attribute 功率/面积含义、read_vcd -time 窗口单位。如脚本选项与安装版本不符，可以修正测量脚本并保留修改 diff；不得改变计算规模、算法或隐瞒失败。

二、功能与周期
只用官方 DW FP32 模型；所有 exp 显式 sig_width=23/exp_width=8/ieee_compliance=0/arch=2。上传包没有测试替代 DW 实现，不得自行用行为替代算子做 PPA。
先进行官方数值回归、有限值检查与协议验证。tb_attention_head_16ctx_4dim 已有独立 dense reference、exp(0)、尾 mask、停顿/预取、复位重用、非法命令和背压测试。默认 TB 和默认 RTL 的 KV_LOAD_PARALLEL 均为16。
默认无停顿预期 C(N)=39*ceil(N/512)+4。N=1152 预期121拍，QK24拍、PV18拍、调度 residual79拍。必须读取实测 MEASURE_END/MAC_SCHEDULE 核对。分母归约与 PV 重叠，不可重复累计墙钟延迟。
使用官方 VCS 先完成默认硬件 N=1..1152 数值/周期扫描，可用 +CONTEXT_FIRST=1 +CONTEXT_LAST=1152；边界协议测试另编译 -pvalue+tb_attention_head_16ctx_4dim.RUN_BOUNDARIES=1 后运行，保留日志。扫描/边界日志不用于功耗窗口。也跑提供的 GQA/cohort TB 做功能回归，但它们的面积不混入主对象。

三、映射与 STA
sources.f 为全部合法 RTL，按默认顶层综合完整设计。不要 ungroup 或跨边界优化 u_mac，亦不要在未映射时 dont_touch 它。
默认有16个 g_context[i]/u_lane/u_mac。它们包括 QK/PV 共享乘法、归约、维度合并及其寄存器/选择门控；其余，包括 query scale、softmax、merge、inverse/output scale 的乘加和 K/V bank，均留在 residual。
完整链接并排除未解析引用、黑盒/未映射算子，保留 check_design、reference/resource、面积和网表证据。
默认实现可能非常大。如果整体 compile 超出服务器内存，可在保持当前接口/边界/规模下采用分层映射再完整链接，报告实际采用的脚本和差异。不能用缩小 CONTEXT_PARALLEL/TILE_SIZE 的结果或单算子线性外推冒充完整结果；完整网表未完成就保留 PENDING。
STA 必须输出全设计 setup/hold WNS、TNS、违例计数、关键路径、max slew/load/cap 检查、未测试检查和约束覆盖。1ns未通过时，121ns只能标 nominal。可另找保留同一接口约束的较慢检查点，但 setup-only 不算时序闭合；不得直接用 WNS/单算子 delay 乘周期数伪造有效 latency。若需要修改 RTL 才能闭合，保留本次失败基线并明确后续工作，不私自把周期数沿用到流水化新设计。
可以进行保持 RTL 功能/周期不变的标准单元映射、扇出缓冲和 hold 缓冲修复；保存修复前后报告与完整最终网表。若采用较慢周期，另建 run-dir，重跑对应时钟下的仿真活动与功耗，不能把1ns功率直接当作慢时钟能量。

四、同一网表的面积、能量分项
报告 TOTAL 与16个 u_mac 子树的面积。根据本安装版本属性确认层次面积是否是所有叶标准单元之和，并与 reference 计数×库面积复核；避免把层次和叶单元重复相加。输出 exact MAC instance list、网表 SHA256 和库单位证据。没有宏则 macro_area=0。
N=1152 的能量至少测 seeds=20261023,20261024,20261025；另外 N=128,512,513,1024 也使用相同硬件测量，不能插值补成 measured。
使用非零 Q/K/V 活动，读取每条 MEASURE_BEGIN/END 标注的命令握手到首次 o_valid 窗口；从 VCD 读取该窗口，核对实际注释时间与周期，不含复位/中止/空闲/输出背压。TOTAL 和每个 MAC 子树必须用同一完整窗口，不能仅测 MAC 活跃阶段。
PrimePower 报告 internal/switching/leakage/total，保留原单位并换为 W。层次网络开关按驱动者计费，边界/时钟/上层缓冲不得重复分配。将 MAC 子树聚合与全局报告核对，保存层次及各 u_mac 报告。报告主输入、时序节点及全网直接/传播/默认/未标注活动覆盖，标注 slew/load 超库表范围、零延迟 RTL 活动无门级毛刺、无 CTS/SPEF 等限制。
E_dynamic_pJ=(P_internal_W+P_switching_W)*window_ns*1000；E_leakage_pJ=P_leakage_W*window_ns*1000。总能量是二者之和。时序/电气失败点只能保留诊断能量，valid energy 保持缺失。

五、可用入口（先核对 site.json 与工具命令）
python3 synopsys/run_flow.py --stage sim --contexts 128,512,513,1024,1152
python3 synopsys/run_flow.py --stage dc
python3 synopsys/run_flow.py --stage sta
python3 synopsys/run_flow.py --stage power
python3 synopsys/collect_candidate.py results/smic28_1ns

run_flow 生成的候选数据尚未审计。对照原始证据填写 measurement_nN.reviewed.json：仅在证据支持时设置 checks=true；缺少的数字保留 null/PENDING。更改 RTL/库/时钟必须使用新 run-dir，不能混用旧网表与活动。不同 N 可共享同一映射面积，但功耗必须来自各自真实活动窗口。
执行：
python3 synopsys/subtract_mac.py results/smic28_1ns/measurement_n1152.reviewed.json --output results/smic28_1ns/nonmac_n1152.json

按计量契约给出 Aresidual=Atotal−sum(Amac)，Eresidual=Etotal−sum(Emac)；动态/漏电分别相减。延迟 residual=(Ctotal−Cqk−Cpv)*T，是调度余项，不是独立 non-MAC critical path。不能按42/121分摊面积或能量，不能减另一份独立综合 MAC 的面积/功耗。KV寄存器和共享控制仍在 residual，应单列说明；若能分离 K/V bank 的面积/能量则额外提供，避免以后与模拟器 SRAM 重复计费。

六、交付
生成 RESULTS_SUMMARY.md 和 machine-readable reviewed JSON / nonmac JSON，列出 TOTAL / MAC / residual 的周期、nominal_ns、valid_ns、area_um2、dynamic_pj、leakage_pj、total_pj。N逐项列出，能量列3种子的 mean/min/max，不能把不同N或head范围混成平均。
同时返回 source SHA256/改动 diff、实际脚本/约束、库/工具版本、网表 SHA256、完整网表（或分层网表及完整链接证明）、关键 STA/电气/面积/功耗/活动覆盖报告、VCS数值和窗口日志、mac_instances.txt、windows.json、失败/缺项清单。可将巨大网表单独压缩或保留服务器路径，但本地相减所需的 per-MAC 数值/报告必须一起返回。
不完整数据不强行写成 MEASURED，不自动回写 SIM-1023。请继续执行已授权的只读检查和测量脚本修正，直到得到可复核结果或明确的工具/资源阻塞。
```
