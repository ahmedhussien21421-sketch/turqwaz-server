const express = require('express');
const http = require('http');
const { Server } = require('socket.io');
const path = require('path');
const store = require('./lib/store');

const app = express();
const server = http.createServer(app);
const io = new Server(server);

app.use(express.json());
app.use(express.urlencoded({ extended: true }));
app.use(express.static(path.join(__dirname, 'public')));

app.get('/api/menu', (req, res) => {
    try {
        const menu = store.getMenu();
        res.json(menu);
    } catch (err) {
        res.status(500).json({ error: 'Failed to load menu' });
    }
});

app.get('/api/orders', (req, res) => {
    try {
        const orders = store.getOrders();
        res.json(orders);
    } catch (err) {
        res.status(500).json({ error: 'Failed to load orders' });
    }
});

app.post('/api/orders', (req, res) => {
    try {
        const orderData = req.body;
        const newOrder = store.addOrder({
            ...orderData,
            status: 'pending',
            createdAt: new Date().toISOString()
        });
        io.emit('new-order', newOrder);
        res.status(201).json({ success: true, order: newOrder });
    } catch (err) {
        res.status(500).json({ error: 'Failed to create order' });
    }
});

app.post('/api/orders/:id/status', (req, res) => {
    try {
        const orderId = req.params.id;
        const { status, prepTime, adminName } = req.body;
        const updatedOrder = store.updateOrderStatus(orderId, { status, prepTime, adminName });
        
        if (!updatedOrder) {
            return res.status(404).json({ error: 'Order not found' });
        }

        io.emit('order-status-updated', updatedOrder);
        res.json({ success: true, order: updatedOrder });
    } catch (err) {
        res.status(500).json({ error: 'Failed to update order status' });
    }
});

io.on('connection', (socket) => {
    console.log('Client/Admin connected:', socket.id);
});

const PORT = process.env.PORT || 3000;
server.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`);
});#
{
  "name": "turqwaz-server",
  "version": "1.0.0",
  "description": "Restaurant Server with Live Audio Alert and Multi-Admin Support",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.19.2",
    "socket.io": "^4.7.5"
  },
  "engines": {
    "node": ">=18.0.0"
  }
} 
const fs = require('fs');
const path = require('path');

const ORDERS_FILE = path.join(__dirname, '../data_orders.json');
const MENU_FILE = path.join(__dirname, '../menu.json');

function getMenu() {
    if (!fs.existsSync(MENU_FILE)) return [];
    const data = fs.readFileSync(MENU_FILE, 'utf8');
    return JSON.parse(data);
}

function getOrders() {
    if (!fs.existsSync(ORDERS_FILE)) return [];
    try {
        const data = fs.readFileSync(ORDERS_FILE, 'utf8');
        return JSON.parse(data);
    } catch (e) {
        return [];
    }
}

function saveOrders(orders) {
    fs.writeFileSync(ORDERS_FILE, JSON.stringify(orders, null, 2));
}

function addOrder(order) {
    const orders = getOrders();
    const newOrder = { id: Date.now().toString(), ...order };
    orders.unshift(newOrder);
    saveOrders(orders);
    return newOrder;
}

function updateOrderStatus(id, updateData) {
    const orders = getOrders();
    const order = orders.find(o => o.id === id);
    if (!order) return null;

    order.status = updateData.status;
    if (updateData.prepTime) order.prepTime = updateData.prepTime;
    if (updateData.adminName) order.handledBy = updateData.adminName;
    
    saveOrders(orders);
    return order;
}

module.exports = { getMenu, getOrders, addOrder, updateOrderStatus };
<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>لوحة تحكم الإدارة - المطعم</title>
    <style>
        body { font-family: Tahoma, sans-serif; background: #f4f7f6; margin: 0; padding: 20px; }
        .admin-header { background: #fff; padding: 15px 25px; border-radius: 8px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; }
        #orders-container { display: grid; grid-template-columns: repeat(auto-fill, minmax(320px, 1fr)); gap: 20px; }
        .order-card { background: #fff; border-radius: 8px; padding: 15px; box-shadow: 0 2px 8px rgba(0,0,0,0.1); border-right: 5px solid #007bff; }
        .order-card.pending { border-right-color: #ffc107; }
        .order-card.accepted { border-right-color: #28a745; }
        .order-card.rejected { border-right-color: #dc3545; }
        .order-header { display: flex; justify-content: space-between; border-bottom: 1px solid #eee; padding-bottom: 8px; margin-bottom: 10px; font-size: 14px; color: #666; }
        .order-body p { margin: 6px 0; font-size: 14px; }
        .order-items-list { margin: 5px 0 15px 20px; padding: 0; font-size: 13px; color: #444; }
        .total-price { color: #d9534f; font-weight: bold; font-size: 16px; }
        .prep-input { width: 100%; padding: 8px; margin-bottom: 10px; border: 1px solid #ccc; border-radius: 4px; box-sizing: border-box; }
        .btn-group { display: flex; gap: 10px; }
        .btn-accept, .btn-reject { flex: 1; padding: 10px; border: none; border-radius: 4px; cursor: pointer; font-weight: bold; color: #fff; }
        .btn-accept { background: #28a745; }
        .btn-reject { background: #dc3545; }
        .status-badge { padding: 8px; border-radius: 4px; text-align: center; font-weight: bold; font-size: 13px; background: #e2e3e5; color: #383d41; }
        #login-modal { position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0,0,0,0.7); display: flex; justify-content: center; align-items: center; z-index: 9999; }
        .login-box { background: #fff; padding: 30px; border-radius: 10px; width: 350px; text-align: center; box-shadow: 0 4px 15px rgba(0,0,0,0.2); }
        .login-box select, .login-box input { width: 100%; padding: 10px; margin: 10px 0; border: 1px solid #ccc; border-radius: 5px; box-sizing: border-box; }
        .login-box button { background: #007bff; color: white; border: none; padding: 10px 20px; width: 100%; border-radius: 5px; cursor: pointer; font-weight: bold; font-size: 16px; }
    </style>
</head>
<body>

    <div id="login-modal">
        <div class="login-box">
            <h3>تسجيل دخول الإدارة</h3>
            <p>اختر صفك أو اسمك:</p>
            <select id="admin-role-select">
                <option value="المدير العام">المدير العام</option>
                <option value="كاشير رقم 1">كاشير رقم 1</option>
                <option value="كاشير رقم 2">كاشير رقم 2</option>
            </select>
            <input type="text" id="admin-custom-name" placeholder="أو اكتب اسمك الصريح هنا...">
            <button onclick="setAdminIdentity()">دخول للوحة التحكم</button>
        </div>
    </div>

    <div class="admin-header">
        <h2>🍔 لوحة متابعة الطلبات الحية</h2>
        <div>المسؤول الحالي: <span id="current-admin-display" style="color: #007bff; font-weight: bold;">غير محدد</span></div>
    </div>

    <div id="orders-container"></div>

    <script src="/socket.io/socket.io.js"></script>
    <script src="admin.js"></script>
</body>
</html> 
const socket = io();
let orders = [];
let currentAdminName = localStorage.getItem('restaurant_admin_name') || '';

function setAdminIdentity() {
    const select = document.getElementById('admin-role-select').value;
    const custom = document.getElementById('admin-custom-name').value.trim();
    currentAdminName = custom !== '' ? custom : select;
    localStorage.setItem('restaurant_admin_name', currentAdminName);
    document.getElementById('current-admin-display').innerText = currentAdminName;
    document.getElementById('login-modal').style.display = 'none';
}

window.addEventListener('DOMContentLoaded', () => {
    if (currentAdminName) {
        document.getElementById('current-admin-display').innerText = currentAdminName;
        document.getElementById('login-modal').style.display = 'none';
    }
});

let audioCtx = null;
let ringInterval = null;

function playRingTone() {
    if (ringInterval) return;
    if (!audioCtx) {
        audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    }
    function beep() {
        if (!audioCtx) return;
        if (audioCtx.state === 'suspended') { audioCtx.resume(); }
        playTone(440, 0.2, 0);
        playTone(480, 0.2, 0.25);
    }
    function playTone(freq, duration, delay) {
        setTimeout(() => {
            if (!audioCtx) return;
            const osc = audioCtx.createOscillator();
            const gain = audioCtx.createGain();
            osc.type = 'sine';
            osc.frequency.value = freq;
            gain.gain.setValueAtTime(0.3, audioCtx.currentTime);
            gain.gain.exponentialRampToValueAtTime(0.0001, audioCtx.currentTime + duration);
            osc.connect(gain);
            gain.connect(audioCtx.destination);
            osc.start();
            osc.stop(audioCtx.currentTime + duration);
        }, delay * 1000);
    }
    beep();
    ringInterval = setInterval(beep, 2000);
}

function stopRingTone() {
    if (ringInterval) {
        clearInterval(ringInterval);
        ringInterval = null;
    }
}

async function fetchOrders() {
    try {
        const res = await fetch('/api/orders');
        orders = await res.json();
        renderOrders();
    } catch (err) {
        console.error('Error fetching orders:', err);
    }
}

function renderOrders() {
    const container = document.getElementById('orders-container');
    if (!container) return;

    if (orders.length === 0) {
        container.innerHTML = '<p style="grid-column: 1/-1; text-align: center; color: #777;">لا توجد طلبات جديدة حالياً...</p>';
        return;
    }

    let hasPending = false;
    container.innerHTML = orders.map(order => {
        const isPending = order.status === 'pending';
        if (isPending) hasPending = true;

        let itemsHtml = order.items.map(i => `<li>${i.name} (×${i.quantity}) - ${i.price} د.ع</li>`).join('');

        return `
            <div class="order-card ${order.status}" id="order-${order.id}">
                <div class="order-header">
                    <span>طلب #${order.id.slice(-4)}</span>
                    <span>${new Date(order.createdAt).toLocaleTimeString('ar-IQ')}</span>
                </div>
                <div class="order-body">
                    <p><strong>الزبون:</strong> ${order.customerName || 'بدون اسم'}</p>
                    <p><strong>الهاتف:</strong> <a href="tel:${order.customerPhone}" style="color: #007bff;">📞 ${order.customerPhone}</a></p>
                    <p><strong>العنوان:</strong> 📍 ${order.customerAddress}</p>
                    <p><strong>ملاحظات:</strong> <span style="color: #d9534f;">${order.customerNotes || 'لا توجد'}</span></p>
                    <hr style="border: 0; border-top: 1px solid #eee;">
                    <p><strong>الأطباق:</strong></p>
                    <ul class="order-items-list">${itemsHtml}</ul>
                    <p><strong>المجموع:</strong> <span class="total-price">${order.totalPrice} د.ع</span></p>
                </div>
                <div class="order-footer">
                    ${isPending ? `
                        <input type="text" id="prep-time-${order.id}" placeholder="وقت التحضير (مثال: 15 دقيقة)" class="prep-input">
                        <div class="btn-group">
                            <button onclick="updateOrderStatus('${order.id}', 'accepted')" class="btn-accept">قبول</button>
                            <button onclick="updateOrderStatus('${order.id}', 'rejected')" class="btn-reject">رفض</button>
                        </div>
                    ` : `
                        <div class="status-badge">
                            ${order.status === 'accepted' ? '✅ تم القبول' : '❌ مرفوض'} 
                            ${order.prepTime ? `<br>وقت التحضير: ${order.prepTime}` : ''}
                            ${order.handledBy ? `<br><small>بواسطة: ${order.handledBy}</small>` : ''}
                        </div>
                    `}
                </div>
            </div>
        `;
    }).join('');

    if (hasPending) { playRingTone(); } else { stopRingTone(); }
}

async function updateOrderStatus(orderId, status) {
    if (!currentAdminName) {
        alert('الرجاء تحديد اسم المسؤول أولاً!');
        document.getElementById('login-modal').style.display = 'flex';
        return;
    }
    const prepInput = document.getElementById(`prep-time-${orderId}`);
    const prepTime = prepInput ? prepInput.value : '';
    if (status === 'accepted' && !prepTime) {
        alert('يرجى كتابة وقت التحضير!');
        return;
    }
    try {
        await fetch(`/api/orders/${orderId}/status`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ status, prepTime, adminName: currentAdminName })
        });
        stopRingTone();
    } catch (err) {
        console.error('Error updating status:', err);
    }
}

socket.on('new-order', (newOrder) => {
    orders.unshift(newOrder);
    renderOrders();
});

socket.on('order-status-updated', (updatedOrder) => {
    const index = orders.findIndex(o => o.id === updatedOrder.id);
    if (index !== -1) orders[index] = updatedOrder;
    renderOrders();
});

window.onload = () => {
    fetchOrders();
    document.body.addEventListener('click', () => {
        if (audioCtx && audioCtx.state === 'suspended') { audioCtx.resume(); }
    }, { once: true });
};
 turqwaz-server
