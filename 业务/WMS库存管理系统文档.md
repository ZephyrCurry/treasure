# WMS库存管理系统文档

## 1. 项目概述

WMS库存管理系统是一个基于Spring Boot的仓库管理系统，主要负责库存的入库、出库、库内操作和查询等功能。系统采用模块化设计，分为客户端（client）和服务端（server）两个模块。

### 1.1 项目结构

```
wms-inventory/
├── wms-inventory-client/          # 客户端模块，提供接口定义和DTO
│   ├── src/main/java/
│   │   └── com/flyfish/wms/inventory/client/
│   │       ├── annotation/        # 注解定义
│   │       ├── constant/          # 常量定义
│   │       ├── dto/               # 数据传输对象
│   │       ├── entity/            # 实体类
│   │       ├── interfaces/       # 客户端接口
│   │       └── valueobject/       # 值对象
│   └── src/main/resources/
│       └── i18n/                  # 国际化资源文件
├── wms-inventory-server/          # 服务端模块，实现业务逻辑
│   ├── src/main/java/
│   │   └── com/flyfish/wms/inventory/server/
│   │       ├── Application.java   # 启动类
│   │       ├── domain/            # 领域层
│   │       ├── interfaces/        # 接口层
│   │       │   ├── controller/   # 控制器
│   │       │   └── mapper/        # MyBatis映射器
│   │       └── service/           # 服务层
│   └── src/main/resources/
│       └── application.properties # 配置文件
└── stages/                         # CI/CD配置
```

### 1.2 技术栈

- **框架**: Spring Boot 2.2.5
- **数据库**: MyBatis Plus
- **Java版本**: 1.8
- **依赖管理**: Maven
- **事务管理**: Spring Transaction
- **幂等性**: 基于注解的幂等性控制

### 1.3 核心实体

#### FwInventory（库存表）
- **主键**: id
- **维度字段**: sku, warehouseCode, platformCode, locationCode, containerCode, poNo, status, quality
- **数量字段**: availableQty（可用数）, expShelveQty（待上架数）, lockingQty（锁定数）, occupiedQty（占用数）, suspenseQty（冻结数）, inTransitQty（在途数）
- **其他字段**: dimSum（库存唯一标识）, firstPutawayTime, lastPutawayTime, lastOutboundTime

#### FwInventoryDetail（库存明细表）
- 记录每个工单（dispatchNo）的库存明细信息
- 关联到FwInventory，记录库存状态变化

#### FwInventoryLog（库存流水表）
- 记录所有库存变动流水
- 包含业务类型、业务单号、变动数量等信息

## 2. 核心概念

### 2.1 库存状态（InventoryStatus）

| 状态码 | 状态名称 | 说明 |
|--------|---------|------|
| 100 | 待入库 | 货物已到达仓库，等待入库 |
| 200 | 待上架 | 已入库，等待上架到库位 |
| 201 | 待移库上架 | 移库操作中，等待上架 |
| 500 | 已上架 | 已上架到库位，可用 |
| 700 | 已下架 | 已从库位下架，准备出库 |
| 800 | 已分拣 | 已分拣完成，等待出库 |
| 900 | 不可用 | 库存不可用 |

### 2.2 库存明细状态（InventoryDetailStatus）

| 状态码 | 状态名称 | 说明 |
|--------|---------|------|
| 100 | 在途 | 货物在运输途中 |
| 200 | 待上架 | 等待上架 |
| 500 | 可用 | 库存可用 |
| 600 | 锁定 | 库存已锁定，准备出库 |
| 700 | 占用 | 库存被占用 |
| 900 | 冻结 | 库存被冻结 |

### 2.3 业务类型（BusinessType）

系统支持多种业务类型，包括：
- **入库相关**: INBOUND（入库）、PUTAWAY（上架）、EXCEPTION_PUTAWAY（异常上架）
- **出库相关**: LOCK（锁库）、PICKUP（拣货）、SORTING（分拣）、OUTBOUND（出库）
- **库内操作**: MOVE_DOWNSHELF（移库下架）、MOVE_PUTAWAY（移库上架）、DIRECT_MOVE（直接移库）
- **库存调整**: ADJUST_ADD（盘盈）、ADJUST_SUB（盘亏）
- **异常处理**: PICKUP_EXCEPTION_FREEZE（拣货异常冻结）、PACK_EXCEPTION_FREEZE（打包异常冻结）

## 3. 入库流程

### 3.1 流程概述

入库流程包括三个主要步骤：
1. **入库（Inbound）**: 货物到达仓库，创建待上架库存
2. **上架（Putaway）**: 将待上架库存上架到指定库位，变为可用库存
3. **异常上架（Exception Putaway）**: 异常情况下，将待上架库存上架为冻结状态

### 3.2 入库接口

#### 3.2.1 入库（Inbound）

**接口路径**: `POST /api/v1/inbound`

**Controller**: `InboundController.inbound()`

**Service**: `InboundService.inbound()`

**功能说明**:
- 创建待上架库存（状态：EXP_SHELVE）
- 增加待上架数量（expShelveQty）
- 创建库存明细记录（状态：EXP_SHELVE）
- 更新库位库龄信息

**请求参数**（InboundRequest）:
- `warehouseCode`: 仓库编码
- `platformCode`: 平台编码
- `sku`: SKU编码
- `containerCode`: 容器编码
- `qty`: 数量
- `quality`: 货品等级（100:正常, 200:二手）
- `poNo`: 采购单号
- `businessNo`: 业务单号
- `businessCode`: 业务流水号
- `dispatchNos`: 工单号列表

**库存变化**:
- 从无到有创建库存记录
- `expShelveQty` += qty
- 库存状态：EXP_SHELVE（200）

#### 3.2.2 上架（Putaway）

**接口路径**: `POST /api/v1/putaway`

**Controller**: `InboundController.putaway()`

**Service**: `InboundService.putaway()`

**功能说明**:
- 减少容器中的待上架数量
- 增加库位中的可用数量
- 更新库存明细状态：EXP_SHELVE → AVAILABLE
- 触发上架回调（PutawayCallback）

**请求参数**（PutawayRequest）:
- `warehouseCode`: 仓库编码
- `platformCode`: 平台编码
- `sku`: SKU编码
- `containerCode`: 原容器编码（待上架库存所在容器）
- `locationCode`: 目标库位编码
- `qty`: 数量
- `quality`: 货品等级
- `poNo`: 采购单号
- `businessNo`: 业务单号
- `businessCode`: 业务流水号
- `dispatchNos`: 工单号列表

**库存变化**:
1. 容器库存（待上架）:
   - `expShelveQty` -= qty
   - 库存状态：EXP_SHELVE（200）

2. 库位库存（可用）:
   - `availableQty` += qty
   - 库存状态：SHELVED（500）

#### 3.2.3 异常上架（Exception Putaway）

**接口路径**: `POST /api/v1/exceptionPutaway`

**Controller**: `InboundController.exceptionPutaway()`

**Service**: `InboundService.exceptionPutaway()`

**功能说明**:
- 减少容器中的待上架数量
- 增加库位中的冻结数量
- 更新库存明细状态：EXP_SHELVE → SUSPENSE
- 触发上架回调

**请求参数**（ExceptionPutawayRequest）:
- 与PutawayRequest类似，但不包含poNo

**库存变化**:
1. 容器库存（待上架）:
   - `expShelveQty` -= qty

2. 库位库存（冻结）:
   - `suspenseQty` += qty
   - 库存状态：SHELVED（500）

### 3.3 入库流程图

```
货物到达
   ↓
[入库接口] → 创建待上架库存（容器中，状态：待上架）
   ↓
[上架接口] → 容器待上架 → 库位可用库存（状态：已上架）
   ↓
库存可用，可以出库
```

## 4. 出库流程

### 4.1 流程概述

出库流程包括多个步骤：
1. **锁库（Lock）**: 锁定可用库存，准备出库
2. **拣货（Pickup）**: 从库位拣货到容器
3. **分拣（Sorting）**: 在容器间分拣
4. **出库（Outbound）**: 最终出库，减少库存

### 4.2 出库接口（V1版本）

#### 4.2.1 锁库（Lock）

**接口路径**: `POST /api/v1/outbound/lock`

**Controller**: `OutboundController.lock()`

**Service**: `OutboundService.lock()`

**功能说明**:
- 锁定可用库存，准备出库
- 支持部分锁定（partLocking=true时，锁定实际可用数量）
- 更新库存明细状态：AVAILABLE → LOCKING

**请求参数**（LockRequest）:
- `warehouseCode`: 仓库编码
- `platformCode`: 平台编码
- `sku`: SKU编码
- `locationCode`: 库位编码
- `qty`: 锁定数量
- `quality`: 货品等级
- `poNo`: 采购单号
- `partLocking`: 是否部分锁定
- `businessNo`: 业务单号
- `businessCode`: 业务流水号
- `dispatchNos`: 工单号列表

**返回结果**（LockResponse）:
- `qty`: 实际锁定数量

**库存变化**:
- `availableQty` -= qty
- `lockingQty` += qty
- 库存状态：SHELVED（500）

#### 4.2.2 取消锁库（Cancel Lock）

**接口路径**: `POST /api/v1/outbound/cancelLock`

**Controller**: `OutboundController.cancelLock()`

**Service**: `OutboundService.cancelLock()`

**功能说明**:
- 取消锁定，将锁定库存恢复为可用库存
- 更新库存明细状态：LOCKING → AVAILABLE

**请求参数**（CancelLockRequest）:
- 与LockRequest类似

**库存变化**:
- `lockingQty` -= qty
- `availableQty` += qty

#### 4.2.3 拣货（Pickup）

**接口路径**: `POST /api/v1/outbound/pickup`

**Controller**: `OutboundController.pickup()`

**Service**: `OutboundService.pickup()`

**功能说明**:
- 从库位拣货到容器
- 库位锁定数量减少，容器锁定数量增加
- 更新库存明细状态：AVAILABLE → LOCKING

**请求参数**（PickupRequest）:
- `warehouseCode`: 仓库编码
- `platformCode`: 平台编码
- `sku`: SKU编码
- `fromLocationCode`: 源库位编码
- `toContainerCode`: 目标容器编码
- `qty`: 数量
- `quality`: 货品等级
- `poNo`: 采购单号
- `businessNo`: 业务单号
- `businessCode`: 业务流水号
- `dispatchNos`: 工单号列表

**库存变化**:
1. 库位库存:
   - `lockingQty` -= qty
   - 库存状态：SHELVED（500）

2. 容器库存:
   - `lockingQty` += qty
   - 库存状态：DOWNSHELF（700）

#### 4.2.4 工单拣货（Dispatch Pickup）

**接口路径**: `POST /api/v1/outbound/dispatchPickup`

**Controller**: `OutboundController.dispatchPickup()`

**Service**: `OutboundService.dispatchPickup()`

**功能说明**:
- 按工单拣货，每次拣货1件
- 从库位锁定库存拣货到容器
- 更新库存明细状态：LOCKING → LOCKING

**请求参数**（DispatchPickupRequest）:
- `warehouseCode`: 仓库编码
- `platformCode`: 平台编码
- `sku`: SKU编码
- `fromLocationCode`: 源库位编码
- `toContainerCode`: 目标容器编码
- `quality`: 货品等级
- `dispatchNo`: 工单号
- `businessNo`: 业务单号
- `businessCode`: 业务流水号

**库存变化**:
- 库位 `lockingQty` -= 1
- 容器 `lockingQty` += 1

#### 4.2.5 分拣（Sorting）

**接口路径**: `POST /api/v1/outbound/sorting`

**Controller**: `OutboundController.sorting()`

**Service**: `OutboundService.sorting()`

**功能说明**:
- 在容器间分拣
- 从源容器分拣到目标容器
- 更新库存状态：DOWNSHELF → SORTED

**请求参数**（SortingRequest）:
- `warehouseCode`: 仓库编码
- `platformCode`: 平台编码
- `sku`: SKU编码
- `fromContainerCode`: 源容器编码
- `toContainerCode`: 目标容器编码
- `qty`: 数量
- `quality`: 货品等级
- `poNo`: 采购单号
- `businessNo`: 业务单号
- `businessCode`: 业务流水号
- `dispatchNos`: 工单号列表

**库存变化**:
1. 源容器:
   - `lockingQty` -= qty
   - 库存状态：DOWNSHELF（700）

2. 目标容器:
   - `lockingQty` += qty
   - 库存状态：SORTED（800）

#### 4.2.6 出库（Outbound V3）

**接口路径**: `POST /api/v3/outbound`

**Controller**: `OutboundV3Controller.outboundV3()`

**Service**: `OutboundService.outbound()`

**功能说明**:
- 最终出库，减少库存
- 从已分拣容器中出库
- 更新库存明细状态：LOCKING → 删除明细
- 触发出库回调（OutboundCallback）

**请求参数**（OutboundV3Request）:
- `warehouseCode`: 仓库编码
- `platformCode`: 平台编码
- `sku`: SKU编码
- `containerCode`: 容器编码
- `qty`: 数量
- `quality`: 货品等级
- `poNo`: 采购单号
- `businessNo`: 业务单号
- `businessCode`: 业务流水号
- `dispatchNos`: 工单号列表

**库存变化**:
- 容器 `lockingQty` -= qty
- 库存状态：SORTED（800）
- 如果库存数量为0，可能删除库存记录

### 4.3 异常处理接口

#### 4.3.1 取消打包换箱（Cancel Packing Change Container）

**接口路径**: `POST /api/v1/outbound/cancelPackingChangeContainer`

**功能说明**: 取消打包时换箱，将已分拣库存转回待上架状态

**库存变化**:
- 源容器（SORTED）: `lockingQty` -= qty
- 目标容器（EXP_SHELVE）: `expShelveQty` += qty

#### 4.3.2 播种打包异常冻结（Sorting Packing Exception Freeze）

**接口路径**: `POST /api/v1/outbound/sortingPackingExceptionFreeze`

**功能说明**: 播种打包异常时，将已分拣库存冻结到库位

**库存变化**:
- 源容器（SORTED）: `lockingQty` -= qty
- 目标库位: `suspenseQty` += qty

#### 4.3.3 不播种打包异常冻结（Not Sorting Packing Exception Freeze）

**接口路径**: `POST /api/v1/outbound/notSortingPackingExceptionFreeze`

**功能说明**: 不播种打包异常时，将已下架库存冻结到库位

**库存变化**:
- 源容器（DOWNSHELF）: `lockingQty` -= qty
- 目标库位: `suspenseQty` += qty

#### 4.3.4 拣货异常冻结（Pickup Exception Freeze）

**接口路径**: `POST /api/v1/outbound/pickupExceptionFreeze`

**功能说明**: 拣货异常时，将可用库存冻结到库位

**库存变化**:
- 源库位: `lockingQty` -= qty
- 目标库位: `suspenseQty` += qty

#### 4.3.5 工单拣货异常冻结（Dispatch Pickup Exception Freeze）

**接口路径**: `POST /api/v1/outbound/dispatchPickupExceptionFreeze`

**功能说明**: 工单拣货异常时，将锁定库存冻结到库位

**库存变化**:
- 源库位: `lockingQty` -= 1
- 目标库位: `suspenseQty` += 1

#### 4.3.6 取消拣货换箱（Cancel Pickup Change Container）

**接口路径**: `POST /api/v1/outbound/cancelPickupChangeContainer`

**功能说明**: 取消拣货时换箱，将已下架库存转回待上架状态

**库存变化**:
- 源容器（DOWNSHELF）: `lockingQty` -= qty
- 目标容器（EXP_SHELVE）: `expShelveQty` += qty

#### 4.3.7 确认找回（Confirm Find）

**接口路径**: `POST /api/v1/outbound/comfirmFind`

**功能说明**: 确认找回冻结库存，将冻结库存恢复为可用库存

**库存变化**:
- 源库位: `suspenseQty` -= qty
- 目标库位: `availableQty` += qty

#### 4.3.8 单品越库关箱（Single Cross Close Container）

**接口路径**: `POST /api/v1/outbound/singleCrossCloseContainer`

**功能说明**: 单品越库关箱，创建已下架锁定库存

**库存变化**:
- 容器: `lockingQty` += 1
- 库存状态：DOWNSHELF（700）

#### 4.3.9 面辅料取消拣货（Material Cancel Pickup）

**接口路径**: `POST /api/v1/outbound/materialCancelPickUp`

**功能说明**: 面辅料取消拣货，将已上架锁定库存转回待上架状态

**库存变化**:
- 源库位（SHELVED）: `lockingQty` -= qty
- 目标容器（EXP_SHELVE）: `expShelveQty` += qty

#### 4.3.10 面辅料拣货（Material Pickup）

**接口路径**: `POST /api/v1/outbound/materialPickUp`

**功能说明**: 面辅料拣货，从库位拣货到容器

**库存变化**:
- 源库位: `lockingQty` -= qty
- 目标容器（SORTED）: `lockingQty` += qty

### 4.4 出库流程图

```
可用库存（库位）
   ↓
[锁库接口] → 锁定库存（库位，状态：已上架）
   ↓
[拣货接口] → 锁定库存（容器，状态：已下架）
   ↓
[分拣接口] → 锁定库存（容器，状态：已分拣）
   ↓
[出库接口] → 库存减少/删除
```

## 5. 库内操作

### 5.1 移库操作

#### 5.1.1 移库下架（Move Downshelf）

**接口路径**: `POST /api/v1/inner/moveDownshelf`

**Controller**: `InnerController.moveDownshelf()`

**功能说明**: 移库下架，从源库位下架到容器

**库存变化**:
- 源库位: `availableQty` -= qty
- 目标容器（EXP_MOVE_SHELVE）: `expShelveQty` += qty

#### 5.1.2 移库异常下架（Move Exception Downshelf）

**接口路径**: `POST /api/v1/inner/moveExceptionDownshelf`

**功能说明**: 移库异常下架，从源库位下架到容器（待移库上架状态）

#### 5.1.3 移库上架（Move Putaway）

**接口路径**: `POST /api/v1/inner/movePutaway`

**功能说明**: 移库上架，从容器上架到目标库位

**库存变化**:
- 源容器（EXP_MOVE_SHELVE）: `expShelveQty` -= qty
- 目标库位: `availableQty` += qty

#### 5.1.4 移库异常上架（Move Exception Putaway）

**接口路径**: `POST /api/v1/inner/moveExceptionPutaway`

**功能说明**: 移库异常上架，从容器上架到目标库位（冻结状态）

**库存变化**:
- 源容器: `expShelveQty` -= qty
- 目标库位: `suspenseQty` += qty

#### 5.1.5 移库上架V2（Move Putaway V2）

**接口路径**: `POST /api/v2/inner/movePutawayV2`

**Controller**: `InnerV2Controller.movePutawayV2()`

**功能说明**: V2版本的移库上架接口

#### 5.1.6 移库异常上架V2（Move Exception Putaway V2）

**接口路径**: `POST /api/v2/inner/moveExceptionPutawayV2`

**功能说明**: V2版本的移库异常上架接口

#### 5.1.7 直接移库（Direct Move）

**接口路径**: `POST /api/v1/inner/directMove`

**功能说明**: 直接移库，库位间直接移动

**库存变化**:
- 源库位: `availableQty` -= qty
- 目标库位: `availableQty` += qty

### 5.2 库存调整

#### 5.2.1 盘盈（Adjust Add）

**接口路径**: `POST /api/v1/inner/adjustAdd`

**功能说明**: 盘盈，增加可用库存

**库存变化**:
- 库位: `availableQty` += qty

#### 5.2.2 不可用库位盘盈（Unavailable Adjust Add）

**接口路径**: `POST /api/v1/inner/unavailableAdjustAdd`

**功能说明**: 不可用库位盘盈，增加冻结库存

**库存变化**:
- 库位: `suspenseQty` += qty

#### 5.2.3 盘亏（Adjust Sub）

**接口路径**: `POST /api/v1/inner/adjustSub`

**功能说明**: 盘亏，减少可用库存

**库存变化**:
- 库位: `availableQty` -= qty

### 5.3 库存占用

#### 5.3.1 库存占用（Occupy）

**接口路径**: `POST /api/v1/inner/occupy`

**功能说明**: 占用可用库存

**库存变化**:
- 库位: `availableQty` -= qty
- 库位: `occupiedQty` += qty

#### 5.3.2 取消占用（Cancel Occupy）

**接口路径**: `POST /api/v1/inner/cancelOccupy`

**功能说明**: 取消占用，恢复为可用库存

**库存变化**:
- 库位: `occupiedQty` -= qty
- 库位: `availableQty` += qty

### 5.4 冻结操作

#### 5.4.1 取消冻结（Cancel Freeze）

**接口路径**: `POST /api/v1/inner/cancelFreeze`

**功能说明**: 取消冻结，将冻结库存恢复为可用库存

**库存变化**:
- 库位: `suspenseQty` -= qty
- 库位: `availableQty` += qty

#### 5.4.2 盘点异常冻结（Stock Take Exception Freeze）

**接口路径**: `POST /api/v1/inner/stockTakeExceptionFreeze`

**功能说明**: 盘点异常冻结，将可用库存冻结

**库存变化**:
- 库位: `availableQty` -= qty
- 库位: `suspenseQty` += qty

#### 5.4.3 冻结库存移动（Freeze Inventory Move）

**接口路径**: `POST /api/v1/inner/freezeInventoryMove`

**功能说明**: 不可用库位盘点异常冻结，冻结库存移动

**库存变化**:
- 源库位: `suspenseQty` -= qty
- 目标库位: `suspenseQty` += qty

### 5.5 紧急补货

#### 5.5.1 紧急补货上架（Emergency Replenishment Putaway）

**接口路径**: `POST /api/v1/inner/emergencyReplenishmentPutaway`

**功能说明**: 紧急补货上架，从容器上架到库位

**库存变化**:
- 源容器: `expShelveQty` -= qty
- 目标库位: `availableQty` += qty

## 6. 批量操作

### 6.1 批量库内操作

#### 6.1.1 批量直接移库（Batch Direct Move）

**接口路径**: `POST /api/v1/batchInner/batchDirectMove`

**Controller**: `BatchInnerController.batchDirectMove()`

**Service**: `BatchInnerService.batchDirectMove()`

**功能说明**: 批量直接移库，支持多个SKU批量移库

**请求参数**（BatchDirectMoveRequest）:
- `warehouseCode`: 仓库编码
- `platformCode`: 平台编码
- `fromLocationCode`: 源库位编码
- `toLocationCode`: 目标库位编码
- `items`: 移库明细列表（SKU、数量等）
- `businessNo`: 业务单号
- `businessCode`: 业务流水号

### 6.2 批量出库操作

#### 6.2.1 批量完成出库异常（Batch Complete Outbound Exception）

**接口路径**: `POST /api/v1/batchOutbound/batchCompleteOutboundException`

**Controller**: `BatchOutboundController.batchCompleteOutboundException()`

**Service**: `BatchOutboundService.batchCompleteOutboundException()`

**功能说明**: 批量完成出库异常单

**请求参数**（BatchCompleteOutboundExceptionRequest）:
- `warehouseCode`: 仓库编码
- `platformCode`: 平台编码
- `items`: 异常单明细列表
- `businessNo`: 业务单号
- `businessCode`: 业务流水号

## 7. 查询接口

### 7.1 库存查询（V1版本）

#### 7.1.1 库存分页查询（Query Inventories）

**接口路径**: `POST /api/v1/inventory/queryInventories`

**Controller**: `InventoryQueryController.queryInventories()`

**功能说明**: 分页查询库存信息

**请求参数**（QueryInventoriesRequest）:
- `warehouseCode`: 仓库编码
- `platformCode`: 平台编码
- `sku`: SKU编码
- `locationCode`: 库位编码
- `containerCode`: 容器编码
- `status`: 库存状态
- `quality`: 货品等级
- `pageNo`: 页码
- `pageSize`: 每页大小

**返回结果**（QueryInventoriesResponse）:
- `total`: 总记录数
- `list`: 库存列表

#### 7.1.2 按SKU统计在库库存数（Query After Sku）

**接口路径**: `POST /api/v1/inventory/queryAfterSku`

**功能说明**: 按SKU统计在库库存数量

**请求参数**（QueryAfterSkuRequest）:
- `warehouseCode`: 仓库编码
- `platformCode`: 平台编码
- `skus`: SKU列表

**返回结果**: SKU库存统计列表

#### 7.1.3 查询仓库库存数（Query Inventory Qty）

**接口路径**: `POST /api/v1/inventory/queryInventoryQty`

**功能说明**: 查询仓库库存数量统计

**请求参数**（QueryInventoryQtyRequest）:
- `warehouseCode`: 仓库编码
- `platformCodes`: 平台编码列表

**返回结果**（QueryInventoryQtyResponse）:
- 各类型库存数量统计

#### 7.1.4 库位库存查询（Query Location Inventory）

**接口路径**: `POST /api/v1/inventory/queryLocationInventory`

**功能说明**: 查询指定库位的库存信息

**请求参数**（QueryLocationInventoryRequest）:
- `warehouseCode`: 仓库编码
- `locationCode`: 库位编码
- `pageNo`: 页码
- `pageSize`: 每页大小

**返回结果**（QueryLocationInventoryResponse）:
- 库位库存列表

#### 7.1.5 库存汇总统计（Query Inventory Sku Location）

**接口路径**: `POST /api/v1/inventory/queryInventorySkuLocation`

**功能说明**: 根据SKU、仓库编码、平台编码对库存数据进行汇总

**请求参数**（QueryInventorySkuLocationRequest）:
- `warehouseCode`: 仓库编码
- `platformCode`: 平台编码
- `skus`: SKU列表

**返回结果**: 库存汇总列表（InventoryLocationSum）

#### 7.1.6 库存导出分页查询（Export Inventories）

**接口路径**: `POST /api/v1/inventory/exportInventories`

**功能说明**: 库存导出分页查询

**请求参数**（ExportInventoriesRequest）:
- 与QueryInventoriesRequest类似

**返回结果**（ExportInventoriesResponse）:
- 库存导出数据列表

#### 7.1.7 查询仓库在库库存数和可售SKU数（Query Warehouse Qty）

**接口路径**: `GET /api/v1/inventory/queryWarehouseQty/{warehouseCode}`

**功能说明**: 查询仓库在库库存数和可售SKU数

**路径参数**:
- `warehouseCode`: 仓库编码

**返回结果**: 仓库库存统计列表

#### 7.1.8 查询越库库存（Query Cross Dock Inventory）

**接口路径**: `POST /api/v1/inventory/queryCrossDockInventory`

**功能说明**: 查询12小时后的越库库存

**请求参数**（QueryCrossDockInventoryRequest）:
- `warehouseCode`: 仓库编码
- `platformCode`: 平台编码
- `startInvId`: 起始库存ID
- `areaCodes`: 库区编码列表
- `pageSize`: 每页大小

**返回结果**（QueryCrossDockInventoryResponse）:
- 越库库存列表

### 7.2 库存查询（V2版本）

#### 7.2.1 库存明细查询（Get Inventory）

**接口路径**: `GET /api/v2/inventory/{warehouseCode}/{id}`

**Controller**: `InventoryQueryV2Controller.getInventory()`

**功能说明**: 根据仓库编码和库存ID查询库存明细

**路径参数**:
- `warehouseCode`: 仓库编码
- `id`: 库存ID

**返回结果**（QueryInventoryResponse）:
- 库存明细信息

#### 7.2.2 库存批量查询（Query Inventories）

**接口路径**: `POST /api/v2/inventory/queryInventories`

**功能说明**: 批量查询库存信息

**请求参数**（QueryInventoriesRequest）:
- `warehouseCode`: 仓库编码
- `ids`: 库存ID列表

**返回结果**: 库存列表

### 7.3 库存明细查询

#### 7.3.1 工单查询（Query Dispatch）

**接口路径**: `GET /api/v1/inventoryDetail/queryDispatch/{warehouseCode}/{dispatchNo}`

**Controller**: `InventoryDetailQueryController.queryDispatch()`

**功能说明**: 根据工单号查询库存明细

**路径参数**:
- `warehouseCode`: 仓库编码
- `dispatchNo`: 工单号

**返回结果**（QueryDispatchResponse）:
- 工单库存明细信息

### 7.4 库存流水查询

#### 7.4.1 库存流水分页查询（Query Logs - 已废弃）

**接口路径**: `POST /api/v1/inventoryLog/queryLogs`

**Controller**: `InventoryLogQueryController.queryLogs()`

**功能说明**: 库存流水分页查询（已废弃，使用query接口）

**状态**: @Deprecated

#### 7.4.2 库存流水分页查询（Query）

**接口路径**: `POST /api/v1/inventoryLog/query`

**功能说明**: 库存流水分页查询V2

**请求参数**（QueryPageRequest）:
- `warehouseCode`: 仓库编码
- `platformCode`: 平台编码
- `sku`: SKU编码
- `businessType`: 业务类型
- `businessNo`: 业务单号
- `startTime`: 开始时间
- `endTime`: 结束时间
- `pageNo`: 页码
- `pageSize`: 每页大小

**返回结果**（QueryLogsResponse）:
- 库存流水列表

#### 7.4.3 库存流水分页查询（根据起始ID查询）

**接口路径**: `POST /api/v1/inventoryLog/queryAfter`

**功能说明**: 根据起始ID查询库存流水

**请求参数**（QueryAfterRequest）:
- `warehouseCode`: 仓库编码
- `startId`: 起始ID
- `pageSize`: 每页大小

**返回结果**: 库存流水列表

#### 7.4.4 库存流水分页查询V2（根据起始时间查询）

**接口路径**: `POST /api/v2/inventoryLog/queryAfter`

**Controller**: `InventoryLogQueryV2Controller.queryAfter()`

**功能说明**: 根据起始时间查询库存流水

**请求参数**（QueryAfterV2Request）:
- `warehouseCode`: 仓库编码
- `startTime`: 起始时间
- `pageSize`: 每页大小

**返回结果**: 库存流水列表

### 7.5 库存明细流水查询

#### 7.5.1 库存明细流水查询（Query Detail Logs）

**接口路径**: `POST /api/v1/inventoryDetailLog/queryDetailLogs`

**Controller**: `InventoryDetailLogQueryController.queryDetailLogs()`

**功能说明**: 查询库存明细流水

**请求参数**（QueryDetailLogsRequest）:
- `warehouseCode`: 仓库编码
- `dispatchNo`: 工单号
- `startTime`: 开始时间
- `endTime`: 结束时间

**返回结果**: 库存明细流水列表（FwInventoryDetailLog）

## 8. 所有Controller接口汇总

### 8.1 入库相关接口（InboundController）

| 接口路径 | 方法 | 功能说明 |
|---------|------|---------|
| POST /api/v1/inbound | inbound() | 入库 |
| POST /api/v1/putaway | putaway() | 上架 |
| POST /api/v1/exceptionPutaway | exceptionPutaway() | 异常上架 |

### 8.2 出库相关接口（OutboundController - V1）

| 接口路径 | 方法 | 功能说明 |
|---------|------|---------|
| POST /api/v1/outbound/lock | lock() | 锁库 |
| POST /api/v1/outbound/cancelLock | cancelLock() | 取消锁库 |
| POST /api/v1/outbound/pickup | pickup() | 拣货 |
| POST /api/v1/outbound/dispatchPickup | dispatchPickup() | 工单拣货 |
| POST /api/v1/outbound/sorting | sorting() | 分拣 |
| POST /api/v1/outbound/cancelPackingChangeContainer | cancelPackingChangeContainer() | 取消打包换箱 |
| POST /api/v1/outbound/sortingPackingExceptionFreeze | sortingPackingExceptionFreeze() | 播种打包异常冻结 |
| POST /api/v1/outbound/notSortingPackingExceptionFreeze | notSortingPackingExceptionFreeze() | 不播种打包异常冻结 |
| POST /api/v1/outbound/pickupExceptionFreeze | pickupExceptionFreeze() | 拣货异常冻结 |
| POST /api/v1/outbound/dispatchPickupExceptionFreeze | dispatchPickupExceptionFreeze() | 工单拣货异常冻结 |
| POST /api/v1/outbound/cancelPickupChangeContainer | cancelPickupChangeContainer() | 取消拣货换箱 |
| POST /api/v1/outbound/comfirmFind | comfirmFind() | 确认找回 |
| POST /api/v1/outbound/singleCrossCloseContainer | singleCrossCloseContainer() | 单品越库关箱 |
| POST /api/v1/outbound/materialCancelPickUp | materialCancelPickUp() | 面辅料取消拣货 |
| POST /api/v1/outbound/materialPickUp | materialPickUp() | 面辅料拣货 |

### 8.3 出库相关接口（OutboundV3Controller - V3）

| 接口路径 | 方法 | 功能说明 |
|---------|------|---------|
| POST /api/v3/outbound | outboundV3() | 出库V3 |

### 8.4 库内操作接口（InnerController）

| 接口路径 | 方法 | 功能说明 |
|---------|------|---------|
| POST /api/v1/inner/occupy | occupy() | 库存占用 |
| POST /api/v1/inner/cancelOccupy | cancelOccupy() | 取消占用 |
| POST /api/v1/inner/moveDownshelf | moveDownshelf() | 移库下架 |
| POST /api/v1/inner/moveExceptionDownshelf | moveExceptionDownshelf() | 移库异常下架 |
| POST /api/v1/inner/movePutaway | movePutaway() | 移库上架 |
| POST /api/v1/inner/moveExceptionPutaway | moveExceptionPutaway() | 移库异常上架 |
| POST /api/v1/inner/adjustAdd | adjustAdd() | 盘盈 |
| POST /api/v1/inner/unavailableAdjustAdd | unavailableAdjustAdd() | 不可用库位盘盈 |
| POST /api/v1/inner/adjustSub | adjustSub() | 盘亏 |
| POST /api/v1/inner/directMove | directMove() | 直接移库 |
| POST /api/v1/inner/cancelFreeze | cancelFreeze() | 取消冻结 |
| POST /api/v1/inner/emergencyReplenishmentPutaway | emergencyReplenishmentPutaway() | 紧急补货上架 |
| POST /api/v1/inner/stockTakeExceptionFreeze | stockTakeExceptionFreeze() | 盘点异常冻结 |
| POST /api/v1/inner/freezeInventoryMove | freezeInventoryMove() | 冻结库存移动 |

### 8.5 库内操作接口（InnerV2Controller - V2）

| 接口路径 | 方法 | 功能说明 |
|---------|------|---------|
| POST /api/v2/inner/movePutawayV2 | movePutawayV2() | 移库上架V2 |
| POST /api/v2/inner/moveExceptionPutawayV2 | moveExceptionPutawayV2() | 移库异常上架V2 |

### 8.6 批量库内操作接口（BatchInnerController）

| 接口路径 | 方法 | 功能说明 |
|---------|------|---------|
| POST /api/v1/batchInner/batchDirectMove | batchDirectMove() | 批量直接移库 |

### 8.7 批量出库操作接口（BatchOutboundController）

| 接口路径 | 方法 | 功能说明 |
|---------|------|---------|
| POST /api/v1/batchOutbound/batchCompleteOutboundException | batchCompleteOutboundException() | 批量完成出库异常 |

### 8.8 库存查询接口（InventoryQueryController - V1）

| 接口路径 | 方法 | 功能说明 |
|---------|------|---------|
| POST /api/v1/inventory/queryInventories | queryInventories() | 库存分页查询 |
| POST /api/v1/inventory/queryAfterSku | queryAfterSku() | 按SKU统计在库库存数 |
| POST /api/v1/inventory/queryInventoryQty | queryInventoryQty() | 查询仓库库存数 |
| POST /api/v1/inventory/queryLocationInventory | queryLocationInventory() | 库位库存查询 |
| POST /api/v1/inventory/queryInventorySkuLocation | queryInventorySkuLocation() | 库存汇总统计 |
| POST /api/v1/inventory/exportInventories | exportInventories() | 库存导出分页查询 |
| GET /api/v1/inventory/queryWarehouseQty/{warehouseCode} | queryWarehouseQty() | 查询仓库在库库存数和可售SKU数 |
| POST /api/v1/inventory/queryCrossDockInventory | queryCrossDockInventory() | 查询越库库存 |

### 8.9 库存查询接口（InventoryQueryV2Controller - V2）

| 接口路径 | 方法 | 功能说明 |
|---------|------|---------|
| GET /api/v2/inventory/{warehouseCode}/{id} | getInventory() | 库存明细查询 |
| POST /api/v2/inventory/queryInventories | queryInventories() | 库存批量查询 |

### 8.10 库存明细查询接口（InventoryDetailQueryController）

| 接口路径 | 方法 | 功能说明 |
|---------|------|---------|
| GET /api/v1/inventoryDetail/queryDispatch/{warehouseCode}/{dispatchNo} | queryDispatch() | 工单查询 |

### 8.11 库存流水查询接口（InventoryLogQueryController - V1）

| 接口路径 | 方法 | 功能说明 |
|---------|------|---------|
| POST /api/v1/inventoryLog/queryLogs | queryLogs() | 库存流水分页查询（已废弃） |
| POST /api/v1/inventoryLog/query | query() | 库存流水分页查询 |
| POST /api/v1/inventoryLog/queryAfter | queryAfter() | 库存流水分页查询（根据起始ID） |

### 8.12 库存流水查询接口（InventoryLogQueryV2Controller - V2）

| 接口路径 | 方法 | 功能说明 |
|---------|------|---------|
| POST /api/v2/inventoryLog/queryAfter | queryAfter() | 库存流水分页查询（根据起始时间） |

### 8.13 库存明细流水查询接口（InventoryDetailLogQueryController）

| 接口路径 | 方法 | 功能说明 |
|---------|------|---------|
| POST /api/v1/inventoryDetailLog/queryDetailLogs | queryDetailLogs() | 库存明细流水查询 |

## 9. 核心设计模式

### 9.1 领域驱动设计（DDD）

系统采用领域驱动设计，主要包含以下层次：

- **领域层（Domain）**: 
  - `InventoryUpdater`: 库存更新器
  - `InventoryDetailUpdater`: 库存明细更新器
  - `LocationInventoryAgeUpdater`: 库位库龄更新器
  - `Repository`: 仓储接口

- **应用层（Service）**: 
  - 各种Service类，负责业务流程编排

- **接口层（Controller）**: 
  - RESTful API接口

### 9.2 回调机制

系统使用回调机制处理业务扩展：

- `InventoryUpdateCallback`: 库存更新回调接口
- `PutawayCallback`: 上架回调
- `OutboundCallback`: 出库回调
- `NoOpCallback`: 空操作回调

### 9.3 幂等性控制

所有写操作都使用`@Idempotent`注解保证幂等性：

```java
@Idempotent(key = "#request.businessCode + '_INBOUND'", shardingKey = "#request.warehouseCode")
```

- `key`: 幂等性键，基于业务流水号
- `shardingKey`: 分片键，基于仓库编码

### 9.4 事务管理

所有写操作都使用`@Transactional`注解保证事务一致性：

```java
@Transactional(rollbackFor = Exception.class)
```

## 10. 国际化支持

系统支持多语言国际化，资源文件位于：
- `wms-inventory-client/src/main/resources/i18n/`

支持的语言：
- 中文（zh）
- 英文（en）
- 西班牙语（es）
- 越南语（vi）
- 高棉语（km）

## 11. 部署配置

### 11.1 环境配置

项目包含多个环境的部署配置：
- `dev`: 开发环境
- `test`: 测试环境
- `pre`: 预发布环境
- `prod`: 生产环境

### 11.2 Docker配置

项目包含多个Dockerfile：
- `Dockerfile`: 生产环境镜像
- `Dockerfile-dev`: 开发环境镜像
- `Dockerfile-pre`: 预发布环境镜像
- `Dockerfile-test`: 测试环境镜像

### 11.3 CI/CD配置

Jenkins流水线配置位于`stages/`目录：
- `Jenkinsfile-dev`: 开发环境流水线
- `Jenkinsfile-test`: 测试环境流水线
- `Jenkinsfile-pre`: 预发布环境流水线
- `Jenkinsfile-prod`: 生产环境流水线

## 12. 总结

WMS库存管理系统是一个功能完善的仓库管理系统，主要特点：

1. **模块化设计**: 客户端和服务端分离，便于集成和维护
2. **完整的业务流程**: 覆盖入库、出库、库内操作等全流程
3. **丰富的查询功能**: 支持多种维度的库存查询
4. **异常处理机制**: 完善的异常处理和恢复机制
5. **幂等性保证**: 所有写操作都支持幂等性
6. **国际化支持**: 支持多语言
7. **事务一致性**: 保证数据一致性

系统通过领域驱动设计、回调机制等设计模式，实现了高内聚、低耦合的架构，便于扩展和维护。
