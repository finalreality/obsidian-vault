```mermaid
---
title: Vulkan Arch
---
flowchart TD
  A[应用程序] <==> B[验证层]
  B <==> C[Vulkan]
  C <==> D[驱动程序]
  D <==> F[硬件]
```