# drawpicture

```mermaid
# drawpicture
flowchart TB
  %% =========================
  %% 输入与路径
  %% =========================
  A[用户输入：年终奖年份 year_nzj] --> B[生成 year_str / next_year / jan_str / feb_str]
  B --> C[base_dir = D:\\03-年终奖\\{year_str}年终综合奖\\0名单梳理\\校机关非事编\\]

  %% =========================
  %% 数据源（读取）
  %% =========================
  subgraph S1[数据源读取（原始表）]
    H1[① 在职名单\n校机关非事编在职人员.xls\n字段：RYBH, XM, LXRQ, HTQSRQ] --> H2[data_htz\nRYBH->gzzh\n默认：是否年终奖=否, 备注=空]

    J1[② 12月奖金结果\n{year_str}年12月奖金-结果-编外.xlsx\n字段：gzzh, xm] --> J2[data_jj_htz\nxm->12月奖金名单]

    T1[③ 停薪名单\n{year_str}年校机关非事编停薪人员.xls\n全字段：含 RYBH, XM, zjbh, txrq, bz...] --> T2[data_tx_gz_full]
    T2 --> T3a[data_tx_gz_only_name\n仅 RYBH, XM\n重命名：RYBH->gzzh, XM->xm]
    T2 --> T3b[data_tx_gz\n全字段\n重命名：RYBH->gzzh\n列名小写]
    T3b --> T4[从 zjbh 提取出生日期\n计算 推算年龄（基准：year_nzj-12-31）]

    D1[④ 新系统在职全名单\n新系统在职全名单.xlsx\n字段：工作证号/人员编号, 单位编号, 单位简称] --> D2[data_dwbh_dwjc\n工作证号/人员编号->gzzh\n单位编号->dwbh\n单位简称->dwjc]

    Q1[⑤ 次年1月起薪名单\n{jan_str}校机关合同制起薪名单.xls\n字段：RYBH, XM, QXRQ] --> Q2[data_gzqx_1yue\nRYBH->gzzh\nXM->1月工资起薪名单\nQXRQ->1月工资中起薪日期]

    R1[⑥ 次年2月起薪名单\n{feb_str}校机关非事编起薪名单.xls\n字段：GZZH, XM, QXRQ] --> R2[data_gzqx_2yue\nGZZH->gzzh\nXM->2月工资起薪名单\nQXRQ->2月工资中起薪日期]
  end

  %% =========================
  %% 人员全集构建（concat + 差集兜底）
  %% =========================
  subgraph S2[人员全集构建（候选池）]
    H2 --> C1[concat：在职 + 停薪(仅工号姓名)\ndata_zz_and_lx = data_htz ∪ data_tx_gz_only_name]
    T3a --> C1

    C1 --> K1[datacompy 比对\njoin key: gzzh\n在职+停薪 vs 12月奖金]
    J2 --> K1

    K1 --> U1[df2_unq_rows\n只在12月奖金名单的人\n= data_only_in_12jj]
    K1 --> U2[df1_unq_rows\n只在在职+停薪的人\n= data_only_in_zz_and_lx]

    C1 --> C2[concat 并集补齐\ndata_all = data_zz_and_lx ∪ data_only_in_12jj]
    U1 --> C2
  end

  %% =========================
  %% 信息补全（VLOOKUP/merge）
  %% =========================
  subgraph S3[信息补全（merge / vlookup）]
    C2 --> M1[merge 单位信息\non gzzh\n补 dwbh/dwjc]
    D2 --> M1

    M1 --> M2[merge 12月奖金\non gzzh\n补 12月奖金名单]
    J2 --> M2

    M2 --> M3[merge 次年1月起薪\non gzzh\n补 1月起薪字段]
    Q2 --> M3

    M3 --> M4[merge 次年2月起薪\non gzzh\n补 2月起薪字段]
    R2 --> M4

    %% 关键：停薪信息来源
    M4 --> M5[merge 停薪全信息\non gzzh\n补 zjbh/txrq/bz/推算年龄 等\n（停薪信息权威来源）]
    T4 --> M5
  end

  %% =========================
  %% 资格判定（是否年终奖）
  %% =========================
  subgraph S4[资格判定：是否年终奖]
    M5 --> P1{是否在12月奖金名单？\n12月奖金名单 != 空}
    P1 -- 是 --> P1a[是否年终奖=是\n备注+=在12月奖金名单]
    P1 -- 否 --> P2{停薪备注 bz 含“退休”？}
    P2 -- 是 --> P2a[是否年终奖=是\n备注+=停薪退休]
    P2 -- 否 --> P3{推算年龄>=55 且 txrq 非空？\n（停薪且疑似退休）}
    P3 -- 是 --> P3a[是否年终奖=是\n备注+=停薪且年龄>=55，疑似退休]
    P3 -- 否 --> P4[保持默认\n是否年终奖=否]
  end

  %% =========================
  %% 特例修正 + PRZTM
  %% =========================
  subgraph S5[编码与导出]
    P1a --> X1[编号类别= gzzh[4:6]]
    P2a --> X1
    P3a --> X1
    P4  --> X1

    X1 --> X2[特殊派遣名单修正\n编号类别=64 且 gzzh in 特批集合\n=> 是否年终奖=是; 备注+=派遣]

    X2 --> Y1{计算 PRZTM\n前提：是否年终奖=是}
    Y1 -- 否 --> Y1a[PRZTM=空]
    Y1 -- 是 --> Y2{是否退休优先？\n(bz 或 备注 含“退休”)}
    Y2 -- 是 --> Y2a[PRZTM=23]
    Y2 -- 否 --> Y3{编号类别=62/88 ?}
    Y3 -- 是 --> Y3a[PRZTM=21]
    Y3 -- 否 --> Y4{编号类别=64 ?}
    Y4 -- 是 --> Y4a[PRZTM=22]
    Y4 -- 否 --> Y4b[PRZTM=空]

    Y2a --> Z[导出 Excel\n年终奖名单筛选-校机关非事编_{year_str}.xlsx]
    Y3a --> Z
    Y4a --> Z
    Y1a --> Z
    Y4b --> Z
  end

markdown
