# proposal.2
flowchart LR
  %% === 接入与分发层 ===
  subgraph Edge[接入与分发层]
    R53[Route 53<br/>DNS]
    CF[CloudFront<br/>CDN + 边缘缓存<br/>(WAF 集成)]
    S3Static[S3 静态托管<br/>React 构建产物]
    MWA[React PWA/移动端<br/>离线缓存/自适应]
    R53 --> CF --> S3Static
    MWA <-->|HTTPS/TLS| CF
  end

  %% === 应用与集成层 ===
  subgraph App[应用与集成层（Node.js）]
    APIGW[API Gateway<br/>REST/HTTP]
    Node[ECS/Fargate 或 Lambda 上的<br/>Node.js 服务]
    SQS[SQS 队列<br/>异步请求缓冲]
    STEP[Step Functions<br/>编排/重试/超时]
    EVT[EventBridge/AppFlow<br/>教育机构集成(拉/推)]
    COG[Cognito<br/>OIDC/JWT 鉴权]
    APIGW -->|JWT 校验| Node
    Node -->|异步投递| SQS --> STEP
    EVT <-.对接接口/批量导入 .-> APIGW
  end

  %% === AI 处理层 ===
  subgraph AI[AI 处理与特征]
    FEAT[Feature Store/Redis<br/>特征服务(可选)]
    SGM[SageMaker Endpoint<br/>实时推理]
    LBD[Lambda(特征组装/后处理)]
    S3DL[S3 数据湖<br/>训练/离线特征/日志]
    STEP --> LBD --> SGM
    LBD --> FEAT
    SGM --> LBD
    LBD --> S3DL
  end

  %% === 数据层 ===
  subgraph Data[数据层]
    MDB[MongoDB Atlas/EC2<br/>主库(学习者/路径/进度)]
    ATH[Athena/Glue<br/>ETL/分析(对 S3)]
    LBD --> MDB
    Node --> MDB
    S3DL --> ATH
  end

  %% === 安全与观测 ===
  subgraph SecOps[安全与观测]
    WAF[WAF/Shield<br/>DDoS/速率限制/Bot]
    KMS[KMS/CMK<br/>加密密钥管理]
    SM[Secrets Manager<br/>凭据/连接串]
    VPC[VPC/子网/安全组/私有链接]
    CW[CloudWatch/X-Ray<br/>日志/指标/分布式追踪]
    CT[CloudTrail<br/>审计/合规]
    CF --> WAF
    APIGW --> VPC
    MDB --> VPC
    S3DL --> KMS
    Node --> CW
    APIGW --> CT
  end

  %% === 端到端数据流 ===
  MWA == ① 登录/请求 ==> APIGW
  APIGW == ② JWT/授权(Cognito) ==> COG
  APIGW == ③ 调用后端 ==> Node
  Node == ④ 读/写 ==> MDB
  Node == ⑤ 需要AI时投递 ==> SQS
  SQS == ⑥ 编排 ==> STEP
  STEP == ⑦ 特征组装/推理 ==> LBD ==> SGM
  LBD == ⑧ 结果回写 ==> MDB
  Node == ⑨ 聚合响应 ==> APIGW
  APIGW == ⑩ 返回结果 ==> MWA

  %% === 内容分发与移动优化 ===
  S3Static -. 静态资源/图片/脚本 .-> CF
  MWA -. ServiceWorker/IndexedDB/Retry/Delta Sync .- CF
