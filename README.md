一、DApp核心功能设计（极简但完整）**
1. **基础功能模块**  
   - **商品管理**：上架/下架商品，设置库存、价格（以Pi计价）、多图展示。  
   - **购物车系统**：用户添加商品、修改数量、实时计算总价（链下计算+链上验证）。  
   - **Pi支付集成**：调用Pi SDK完成支付，支持退款逻辑（超时/取消订单）。  
   - **订单追踪**：链上存储订单状态（`Pending`, `Paid`, `Shipped`, `Refunded`）。  
   - **评价系统**：用户对商品评分（链上存哈希，详细评价存IPFS）。  

2. **附加优化功能**  
   - **活动**：限时（基于智能合约时间锁）。  
   - **库存预警**：库存低于阈值时触发链下通知（如Twilio短信）。  
   - **多语言支持**：默认英语，可扩展中文、西班牙语等。  

---

### **二、技术实现路径（Pi公链适配）**
#### **1. 智能合约开发（Solidity++PiVM适配）**
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract PiRetailStore {
    struct Product {
        uint256 id;
        string name;
        uint256 price; // 单位：Pi (1 Pi = 1e6 最小单位)
        uint256 stock;
        string ipfsImageHash;
    }

    struct Order {
        uint256 productId;
        address buyer;
        uint256 quantity;
        uint256 totalPrice;
        OrderStatus status;
    }

    enum OrderStatus { Pending, Paid, Shipped, Refunded }

    mapping(uint256 => Product) public products;
    mapping(bytes32 => Order) public orders;
    address public owner;

    event ProductListed(uint256 productId, string name);
    event OrderCreated(bytes32 orderId, address buyer);

    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }

    constructor() {
        owner = msg.sender;
    }

    // 商品上架（仅店主可调用）
    function listProduct(uint256 id, string memory name, uint256 price, uint256 stock, string memory imageHash) public onlyOwner {
        products[id] = Product(id, name, price, stock, imageHash);
        emit ProductListed(id, name);
    }

    // 用户下单（支付需调用Pi SDK前置校验）
    function createOrder(uint256 productId, uint256 quantity) public {
        Product storage product = products[productId];
        require(product.stock >= quantity, "Insufficient stock");
        
        bytes32 orderId = keccak256(abi.encodePacked(msg.sender, block.timestamp, productId));
        orders[orderId] = Order(productId, msg.sender, quantity, product.price * quantity, OrderStatus.Pending);
        
        product.stock -= quantity;
        emit OrderCreated(orderId, msg.sender);
    }

    // Pi支付成功回调（由Pi Node触发）
    function confirmPayment(bytes32 orderId) public {
        Order storage order = orders[orderId];
        require(order.status == OrderStatus.Pending, "Invalid order status");
        order.status = OrderStatus.Paid;
    }
}
```

#### **2. 前端技术栈（轻量级+Pi Browser兼容）**
- **框架**：React.js + TypeScript（适配Pi Browser的WebView内核）  
- **状态管理**：Jotai（轻量级原子化状态）  
- **Pi SDK集成**：  
  ```javascript
  import { Pi } from '@pinetwork-js/sdk';

  // 初始化Pi SDK
  Pi.init({ version: '2.0', sandbox: true });

  // 支付流程
  const handlePayment = async (orderId, amount) => {
    const payment = await Pi.createPayment({
      amount: amount,
      memo: `Purchase #${orderId}`,
      metadata: { orderId }
    });

    await payment.submit();
    await payment.waitForStatusUpdate((status) => status.txid);
    
    // 调用合约confirmPayment方法
    const contract = new ethers.Contract(contractAddress, abi, signer);
    await contract.confirmPayment(orderId);
  };
  ```

#### **3. 数据存储方案**
- **链上数据**：商品ID、价格、订单核心状态（Pi公链存储）。  
- **链下数据**：商品高清图片（IPFS）、用户评价文本（IPFS+Filecoin）。  
- **缓存层**：使用The Graph索引链上订单数据，加速查询。  

---

### **三、安全与测试关键点**
1. **不可出错的核心环节**  
   - **支付原子性**：确保“链下购物车计算 → Pi支付 → 链上订单状态更新”全程原子化，防止支付成功但订单未更新。  
   - **库存锁**：用户下单时立即锁定库存，支付超时（如15分钟）后自动释放（需合约定时任务支持）。  

2. **测试用例（示例）**  
   | 测试场景                  | 预期结果                     | 工具                |  
   |---------------------------|------------------------------|---------------------|  
   | 并发10用户抢购最后1件商品 | 仅1人成功，其余提示库存不足  | LoadRunner + Ganache|  
   | Pi支付成功后节点宕机      | 订单状态自动恢复为Paid       | Chaos Mesh          |  
   | 恶意用户发送非法Tx数据    | 合约revert并记录异常事件     | MythX Slither       |  

---

### **四、UI思维导图设计**
```plaintext
零售小店DApp UI结构
│
├── 主页 (Home)
│   ├── 搜索栏 (关键词/分类)
│   ├── 商品瀑布流
│   │   ├── 商品卡片 (图片+名称+价格)
│   │   └── “加入购物车”按钮
│   └── 促销横幅（轮播图）
│
├── 购物车 (Cart)
│   ├── 商品列表（图片+数量+单价）
│   ├── 总价显示
│   ├── “编辑数量”按钮
│   └── “Pi支付”按钮（调用SDK）
│
├── 订单 (Orders)
│   ├── 订单状态筛选（全部/待支付/已发货）
│   ├── 订单卡片（订单号+商品缩略图+状态）
│   └── “确认收货”按钮（触发链上状态更新）
│
└── 个人中心 (Profile)
    ├── 钱包地址显示（Pi账户）
    ├── 语言切换
    └── 联系客服（集成Pi Chat）
```

---

### **五、实战演示步骤（无差错脚本）**
1. **环境准备**  
   - 启动本地Pi Testnet节点（使用`pi-node --env=testnet`）。  
   - 部署合约：`forge create --rpc-url pi-testnet Contract.sol --verify`。  

2. **演示流程**  
   ```bash
   # 1. 店主上架商品
   curl -X POST https://dapp-api.example.com/products \
     -H "Content-Type: application/json" \
     -d '{"id": 1, "name": "Organic Coffee", "price": 314159, "stock": 100}'

   # 2. 用户浏览并下单
   # 前端自动调用合约createOrder(1, 2) → 生成orderId=0x123...

   # 3. Pi支付流程（模拟）
   pi-cli pay --to merchant_wallet --amount 628318 --memo "Order 0x123..."

   # 4. 验证链上状态
   cast call contractAddress "orders(0x123...)" --rpc-url pi-testnet
   # 应返回status=Paid

   # 5. 店主发货（更新状态）
   cast send contractAddress "shipOrder(0x123...)" --pri