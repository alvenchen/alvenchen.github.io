# Data Closed-Loop

## Pipiline

```mermaid
graph TD
    A[数据采集: 传感器/终端数据] --> B[传输存储]
    B --> C[清洗标注]
    C --> D[模型训练]
    D --> E[仿真验证]
    E --> F[真实部署]
    F -->|触发新循环| A
```

