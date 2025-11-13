---
title: "PID 控制器"
date: 2025-11-12T21:41:11-08:00
---

# PID



#### P（Proportional）：比例环节

- **核心作用**：**根据当前误差的大小，即时输出控制量**，是系统响应的 “即时驱动力”。
- 数学表达式：
  - 连续时间：*uP*(*t*)=*Kp*⋅*e*(*t*)   【 *e*(*t*)  为当前时刻误差，*Kp* 为比例系数）】
  - 离散时间（采样系统）：*uP*(*k*)=*Kp*⋅*e*(*k*)【*e*(*k*) 为第 *k* 次采样的误差）】
- 控制特性：
  - 误差越大，比例控制量越大，系统响应越快；
  - 比例系数Kp影响：
    - *Kp* 过大：系统响应快，但易超调（超过目标值）、震荡；
    - *Kp* 过小：系统响应迟缓，调节精度低。
- **局限性**：仅比例控制无法消除 “稳态误差”（即系统稳定后，实际值与目标值的偏差，称为 “余差”）。
- **典型场景**：适用于快速响应需求高、允许小余差的系统（如简单的位置控制）。



####  I（Integral）：积分环节

- **核心作用**：**累积过去所有误差的总和**，用于消除比例控制无法解决的 “稳态误差”，实现无余差控制。

- 数学表达式：

  - 连续时间：

  $$
  u_I(t) = K_i \cdot \int_0^t e(\tau) d\tau
  $$

  - 离散时间：
    $$
    u_I(k) = K_i \cdot \sum_{n=0}^k e(n) \cdot T
    $$

    - 离散积分是 “有限个固定宽度（采样周期 T）的矩形面积之和”，由于 T 是常数，可合并到系数中，最终表现为 “误差的累加”。



#### D（Derivative）：微分环节

- **核心作用**：**基于误差的变化率（导数）预测误差趋势**，提前调整控制量，抑制超调、减少震荡，增强系统稳定性。

- 数学表达式：

  - 连续时间：
    $$
    u_D(t) = K_d \cdot \frac{de(t)}{dt}
    $$
    
  - 离散时间：
    $$
    u_D(k) = K_d \cdot \frac{e(k) - e(k-1)}{T}
    $$
    ​	
  
    在实际的离散 PID 控制器中，微分项的计算会把**采样周期 T** 合并到微分系数中。从形式上看，就成了 “当前误差与上一时刻误差的差值” 乘以一个系数 —— 本质上是把 “变化率计算” 中的固定时间间隔 T 融入了系数，所以直观上表现为 “误差相减”。
  
- 控制特性：

  - 微分作用与误差的 “变化速度” 相关：误差变化越快，微分控制量越大（提前干预）；
  - 微分系数Kd影响：
    - \(Kd\) 过大：系统对噪声敏感（噪声变化率大），易产生高频震荡；
    - \(Kd\) 过小：微分作用弱，无法有效抑制超调，系统响应迟缓。

- **典型场景**：大惯性系统（如电机转速控制、机械臂定位），需抑制超调、加快动态响应。

  

## 一、位置式PID



## 二、增量式PID













































































































```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>PID控制器模拟器</title>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <style>
        :root {
            --primary: #3498db;
            --secondary: #2ecc71;
            --accent: #e74c3c;
            --dark: #2c3e50;
            --light: #ecf0f1;
        }
        
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
        }
        
        body {
            background: linear-gradient(135deg, #1a2980, #26d0ce);
            color: var(--light);
            min-height: 100vh;
            padding: 20px;
        }
        
        .container {
            max-width: 1200px;
            margin: 0 auto;
        }
        
        header {
            text-align: center;
            padding: 20px 0;
            margin-bottom: 30px;
        }
        
        h1 {
            font-size: 2.5rem;
            margin-bottom: 10px;
            text-shadow: 0 2px 4px rgba(0,0,0,0.3);
        }
        
        .subtitle {
            font-size: 1.2rem;
            opacity: 0.9;
            max-width: 700px;
            margin: 0 auto;
            line-height: 1.6;
        }
        
        .panel {
            background: rgba(44, 62, 80, 0.8);
            border-radius: 15px;
            padding: 25px;
            box-shadow: 0 10px 30px rgba(0,0,0,0.3);
            backdrop-filter: blur(10px);
            margin-bottom: 30px;
        }
        
        .control-grid {
            display: grid;
            grid-template-columns: 1fr 1fr;
            gap: 25px;
        }
        
        @media (max-width: 768px) {
            .control-grid {
                grid-template-columns: 1fr;
            }
        }
        
        .input-group {
            margin-bottom: 20px;
        }
        
        .input-group label {
            display: block;
            margin-bottom: 8px;
            font-weight: 500;
            font-size: 1.1rem;
        }
        
        .slider-container {
            display: flex;
            align-items: center;
        }
        
        .slider-container input[type="range"] {
            flex: 1;
            height: 8px;
            -webkit-appearance: none;
            background: rgba(236, 240, 241, 0.2);
            border-radius: 4px;
            outline: none;
        }
        
        .slider-container input[type="range"]::-webkit-slider-thumb {
            -webkit-appearance: none;
            width: 20px;
            height: 20px;
            border-radius: 50%;
            background: var(--primary);
            cursor: pointer;
            box-shadow: 0 2px 5px rgba(0,0,0,0.2);
        }
        
        .slider-value {
            width: 60px;
            text-align: center;
            background: rgba(236, 240, 241, 0.1);
            padding: 8px;
            border-radius: 5px;
            margin-left: 15px;
            font-weight: 600;
        }
        
        .chart-container {
            position: relative;
            height: 400px;
            margin-top: 30px;
        }
        
        .button-group {
            display: flex;
            justify-content: center;
            gap: 15px;
            margin-top: 20px;
        }
        
        .btn {
            padding: 12px 30px;
            border: none;
            border-radius: 50px;
            font-size: 1.1rem;
            font-weight: 600;
            cursor: pointer;
            transition: all 0.3s ease;
            box-shadow: 0 4px 8px rgba(0,0,0,0.2);
        }
        
        .btn-primary {
            background: var(--primary);
            color: white;
        }
        
        .btn-primary:hover {
            background: #2980b9;
            transform: translateY(-2px);
        }
        
        .btn-secondary {
            background: var(--secondary);
            color: white;
        }
        
        .btn-secondary:hover {
            background: #27ae60;
            transform: translateY(-2px);
        }
        
        .btn-accent {
            background: var(--accent);
            color: white;
        }
        
        .btn-accent:hover {
            background: #c0392b;
            transform: translateY(-2px);
        }
        
        .pid-info {
            display: grid;
            grid-template-columns: repeat(3, 1fr);
            gap: 20px;
            margin-top: 25px;
        }
        
        @media (max-width: 600px) {
            .pid-info {
                grid-template-columns: 1fr;
            }
        }
        
        .pid-term {
            background: rgba(52, 152, 219, 0.2);
            border-radius: 10px;
            padding: 15px;
            text-align: center;
        }
        
        .pid-term h3 {
            margin-bottom: 10px;
            color: var(--primary);
        }
        
        .pid-value {
            font-size: 1.8rem;
            font-weight: 700;
        }
        
        .system-info {
            display: flex;
            justify-content: space-around;
            margin-top: 20px;
            flex-wrap: wrap;
        }
        
        .info-box {
            background: rgba(46, 204, 113, 0.2);
            border-radius: 10px;
            padding: 15px 25px;
            min-width: 200px;
            text-align: center;
            margin: 10px;
        }
        
        .info-box h3 {
            color: var(--secondary);
            margin-bottom: 10px;
        }
        
        .info-value {
            font-size: 1.5rem;
            font-weight: 700;
        }
        
        .legend {
            display: flex;
            justify-content: center;
            gap: 25px;
            margin-top: 15px;
        }
        
        .legend-item {
            display: flex;
            align-items: center;
            gap: 8px;
        }
        
        .legend-color {
            width: 20px;
            height: 20px;
            border-radius: 3px;
        }
        
        .setpoint-color {
            background-color: #e74c3c;
        }
        
        .process-color {
            background-color: #3498db;
        }
        
        .output-color {
            background-color: #2ecc71;
        }
        
        footer {
            text-align: center;
            margin-top: 30px;
            padding: 20px;
            opacity: 0.7;
            font-size: 0.9rem;
        }
    </style>
</head>
<body>
    <div class="container">
        <header>
            <h1>PID 控制器模拟器</h1>
            <p class="subtitle">通过调整比例(P)、积分(I)和微分(D)参数，实时观察PID控制器对系统响应的调节效果。设定目标值，观察系统如何达到稳定状态。</p>
        </header>
        
        <main>
            <div class="panel">
                <div class="control-grid">
                    <div class="control-section">
                        <h2>PID 参数调整</h2>
                        
                        <div class="input-group">
                            <label for="kp">比例系数 (Kp)</label>
                            <div class="slider-container">
                                <input type="range" id="kp" min="0.1" max="10" step="0.1" value="1.5">
                                <div class="slider-value" id="kp-value">1.5</div>
                            </div>
                        </div>
                        
                        <div class="input-group">
                            <label for="ki">积分系数 (Ki)</label>
                            <div class="slider-container">
                                <input type="range" id="ki" min="0" max="2" step="0.01" value="0.2">
                                <div class="slider-value" id="ki-value">0.2</div>
                            </div>
                        </div>
                        
                        <div class="input-group">
                            <label for="kd">微分系数 (Kd)</label>
                            <div class="slider-container">
                                <input type="range" id="kd" min="0" max="5" step="0.1" value="0.5">
                                <div class="slider-value" id="kd-value">0.5</div>
                            </div>
                        </div>
                        
                        <div class="input-group">
                            <label for="setpoint">设定目标值</label>
                            <div class="slider-container">
                                <input type="range" id="setpoint" min="0" max="100" step="1" value="75">
                                <div class="slider-value" id="setpoint-value">75</div>
                            </div>
                        </div>
                    </div>
                    
                    <div class="control-section">
                        <h2>系统参数</h2>
                        
                        <div class="input-group">
                            <label for="noise">系统噪声水平</label>
                            <div class="slider-container">
                                <input type="range" id="noise" min="0" max="5" step="0.1" value="0.5">
                                <div class="slider-value" id="noise-value">0.5</div>
                            </div>
                        </div>
                        
                        <div class="input-group">
                            <label for="delay">系统延迟 (ms)</label>
                            <div class="slider-container">
                                <input type="range" id="delay" min="0" max="500" step="10" value="100">
                                <div class="slider-value" id="delay-value">100</div>
                            </div>
                        </div>
                        
                        <div class="input-group">
                            <label for="disturbance">扰动幅度</label>
                            <div class="slider-container">
                                <input type="range" id="disturbance" min="0" max="20" step="1" value="5">
                                <div class="slider-value" id="disturbance-value">5</div>
                            </div>
                        </div>
                        
                        <div class="button-group">
                            <button id="disturb-btn" class="btn btn-accent">应用扰动</button>
                        </div>
                    </div>
                </div>
                
                <div class="button-group">
                    <button id="start-btn" class="btn btn-primary">开始模拟</button>
                    <button id="reset-btn" class="btn btn-secondary">重置模拟</button>
                </div>
            </div>
            
            <div class="panel">
                <h2>系统响应曲线</h2>
                <div class="chart-container">
                    <canvas id="responseChart"></canvas>
                </div>
                
                <div class="legend">
                    <div class="legend-item">
                        <div class="legend-color setpoint-color"></div>
                        <span>设定值 (SP)</span>
                    </div>
                    <div class="legend-item">
                        <div class="legend-color process-color"></div>
                        <span>过程变量 (PV)</span>
                    </div>
                    <div class="legend-item">
                        <div class="legend-color output-color"></div>
                        <span>控制器输出 (OUT)</span>
                    </div>
                </div>
            </div>
            
            <div class="panel">
                <h2>实时 PID 参数</h2>
                <div class="pid-info">
                    <div class="pid-term">
                        <h3>比例项 (P)</h3>
                        <div class="pid-value" id="p-term">0.00</div>
                    </div>
                    <div class="pid-term">
                        <h3>积分项 (I)</h3>
                        <div class="pid-value" id="i-term">0.00</div>
                    </div>
                    <div class="pid-term">
                        <h3>微分项 (D)</h3>
                        <div class="pid-value" id="d-term">0.00</div>
                    </div>
                </div>
                
                <div class="system-info">
                    <div class="info-box">
                        <h3>当前误差</h3>
                        <div class="info-value" id="error-value">0.00</div>
                    </div>
                    <div class="info-box">
                        <h3>总积分误差</h3>
                        <div class="info-value" id="integral-value">0.00</div>
                    </div>
                    <div class="info-box">
                        <h3>当前输出</h3>
                        <div class="info-value" id="output-value">0.00</div>
                    </div>
                </div>
            </div>
        </main>
        
        <footer>
            <p>PID控制器模拟器 | 比例-积分-微分控制算法可视化工具 | 通过调整参数优化系统响应</p>
        </footer>
    </div>

    <script>
        // 初始化变量
        let kp = 1.5;
        let ki = 0.2;
        let kd = 0.5;
        let setpoint = 75;
        let noiseLevel = 0.5;
        let systemDelay = 100;
        let disturbance = 5;
        
        let error = 0;
        let prevError = 0;
        let integral = 0;
        let derivative = 0;
        let output = 0;
        let processValue = 0;
        let prevProcessValue = 0;
        
        let simulationRunning = false;
        let simulationInterval;
        let startTime;
        let timeData = [];
        let spData = [];
        let pvData = [];
        let outData = [];
        
        // 获取DOM元素
        const kpSlider = document.getElementById('kp');
        const kiSlider = document.getElementById('ki');
        const kdSlider = document.getElementById('kd');
        const setpointSlider = document.getElementById('setpoint');
        const noiseSlider = document.getElementById('noise');
        const delaySlider = document.getElementById('delay');
        const disturbanceSlider = document.getElementById('disturbance');
        
        const kpValue = document.getElementById('kp-value');
        const kiValue = document.getElementById('ki-value');
        const kdValue = document.getElementById('kd-value');
        const setpointValue = document.getElementById('setpoint-value');
        const noiseValue = document.getElementById('noise-value');
        const delayValue = document.getElementById('delay-value');
        const disturbanceValue = document.getElementById('disturbance-value');
        
        const startBtn = document.getElementById('start-btn');
        const resetBtn = document.getElementById('reset-btn');
        const disturbBtn = document.getElementById('disturb-btn');
        
        const pTerm = document.getElementById('p-term');
        const iTerm = document.getElementById('i-term');
        const dTerm = document.getElementById('d-term');
        const errorValue = document.getElementById('error-value');
        const integralValue = document.getElementById('integral-value');
        const outputValue = document.getElementById('output-value');
        
        // 初始化图表
        const ctx = document.getElementById('responseChart').getContext('2d');
        const responseChart = new Chart(ctx, {
            type: 'line',
            data: {
                labels: [],
                datasets: [
                    {
                        label: '设定值 (SP)',
                        data: [],
                        borderColor: '#e74c3c',
                        backgroundColor: 'rgba(231, 76, 60, 0.1)',
                        borderWidth: 2,
                        fill: false,
                        pointRadius: 0
                    },
                    {
                        label: '过程变量 (PV)',
                        data: [],
                        borderColor: '#3498db',
                        backgroundColor: 'rgba(52, 152, 219, 0.1)',
                        borderWidth: 2,
                        fill: false,
                        pointRadius: 0
                    },
                    {
                        label: '控制器输出 (OUT)',
                        data: [],
                        borderColor: '#2ecc71',
                        backgroundColor: 'rgba(46, 204, 113, 0.1)',
                        borderWidth: 2,
                        fill: false,
                        pointRadius: 0,
                        yAxisID: 'y1'
                    }
                ]
            },
            options: {
                responsive: true,
                maintainAspectRatio: false,
                scales: {
                    x: {
                        title: {
                            display: true,
                            text: '时间 (秒)',
                            color: '#ecf0f1'
                        },
                        grid: {
                            color: 'rgba(236, 240, 241, 0.1)'
                        },
                        ticks: {
                            color: '#ecf0f1'
                        }
                    },
                    y: {
                        title: {
                            display: true,
                            text: '设定值 / 过程变量',
                            color: '#ecf0f1'
                        },
                        min: 0,
                        max: 100,
                        grid: {
                            color: 'rgba(236, 240, 241, 0.1)'
                        },
                        ticks: {
                            color: '#ecf0f1'
                        }
                    },
                    y1: {
                        position: 'right',
                        title: {
                            display: true,
                            text: '控制器输出',
                            color: '#ecf0f1'
                        },
                        min: -10,
                        max: 110,
                        grid: {
                            drawOnChartArea: false,
                            color: 'rgba(236, 240, 241, 0.1)'
                        },
                        ticks: {
                            color: '#ecf0f1'
                        }
                    }
                },
                plugins: {
                    legend: {
                        labels: {
                            color: '#ecf0f1'
                        }
                    },
                    tooltip: {
                        mode: 'index',
                        intersect: false
                    }
                },
                interaction: {
                    mode: 'nearest',
                    axis: 'x',
                    intersect: false
                },
                animation: {
                    duration: 0
                }
            }
        });
        
        // 更新滑块值显示
        function updateSliderValues() {
            kpValue.textContent = kp;
            kiValue.textContent = ki;
            kdValue.textContent = kd;
            setpointValue.textContent = setpoint;
            noiseValue.textContent = noiseLevel;
            delayValue.textContent = systemDelay;
            disturbanceValue.textContent = disturbance;
        }
        
        // 初始化滑块值
        updateSliderValues();
        
        // 事件监听器
        kpSlider.addEventListener('input', function() {
            kp = parseFloat(this.value);
            updateSliderValues();
        });
        
        kiSlider.addEventListener('input', function() {
            ki = parseFloat(this.value);
            updateSliderValues();
        });
        
        kdSlider.addEventListener('input', function() {
            kd = parseFloat(this.value);
            updateSliderValues();
        });
        
        setpointSlider.addEventListener('input', function() {
            setpoint = parseInt(this.value);
            updateSliderValues();
        });
        
        noiseSlider.addEventListener('input', function() {
            noiseLevel = parseFloat(this.value);
            updateSliderValues();
        });
        
        delaySlider.addEventListener('input', function() {
            systemDelay = parseInt(this.value);
            updateSliderValues();
        });
        
        disturbanceSlider.addEventListener('input', function() {
            disturbance = parseInt(this.value);
            updateSliderValues();
        });
        
        startBtn.addEventListener('click', function() {
            if (!simulationRunning) {
                startSimulation();
                startBtn.textContent = '停止模拟';
            } else {
                stopSimulation();
                startBtn.textContent = '开始模拟';
            }
            simulationRunning = !simulationRunning;
        });
        
        resetBtn.addEventListener('click', function() {
            resetSimulation();
        });
        
        disturbBtn.addEventListener('click', function() {
            applyDisturbance();
        });
        
        // 应用干扰
        function applyDisturbance() {
            if (simulationRunning) {
                processValue += disturbance * (Math.random() > 0.5 ? 1 : -1);
            }
        }
        
        // 启动模拟
        function startSimulation() {
            startTime = new Date().getTime();
            simulationInterval = setInterval(updateSimulation, 50);
        }
        
        // 停止模拟
        function stopSimulation() {
            clearInterval(simulationInterval);
        }
        
        // 重置模拟
        function resetSimulation() {
            stopSimulation();
            simulationRunning = false;
            startBtn.textContent = '开始模拟';
            
            // 重置变量
            error = 0;
            prevError = 0;
            integral = 0;
            derivative = 0;
            output = 0;
            processValue = 0;
            prevProcessValue = 0;
            
            // 清空数据
            timeData = [];
            spData = [];
            pvData = [];
            outData = [];
            
            // 重置图表
            responseChart.data.labels = [];
            responseChart.data.datasets[0].data = [];
            responseChart.data.datasets[1].data = [];
            responseChart.data.datasets[2].data = [];
            responseChart.update();
            
            // 更新显示值
            updateDisplayValues();
        }
        
        // 更新模拟
        function updateSimulation() {
            const currentTime = (new Date().getTime() - startTime) / 1000;
            
            // 计算PID
            prevError = error;
            error = setpoint - processValue;
            
            // 积分项（防饱和）
            integral += error * 0.05;
            integral = Math.max(Math.min(integral, 100), -100);
            
            // 微分项
            derivative = (error - prevError) / 0.05;
            
            // 计算输出
            const p = kp * error;
            const i = ki * integral;
            const d = kd * derivative;
            output = p + i + d;
            
            // 限制输出在0-100范围内
            output = Math.max(0, Math.min(100, output));
            
            // 更新过程值（模拟一阶惯性系统）
            const timeConstant = 1.5 + (systemDelay / 1000);
            const alpha = 1 - Math.exp(-0.05 / timeConstant);
            processValue += alpha * (output - processValue);
            
            // 添加噪声
            processValue += (Math.random() - 0.5) * noiseLevel;
            
            // 记录数据
            if (timeData.length > 200) {
                timeData.shift();
                spData.shift();
                pvData.shift();
                outData.shift();
            }
            
            timeData.push(currentTime.toFixed(1));
            spData.push(setpoint);
            pvData.push(processValue);
            outData.push(output);
            
            // 更新图表
            responseChart.data.labels = timeData;
            responseChart.data.datasets[0].data = spData;
            responseChart.data.datasets[1].data = pvData;
            responseChart.data.datasets[2].data = outData;
            responseChart.update();
            
            // 更新显示值
            updateDisplayValues();
        }
        
        // 更新显示值
        function updateDisplayValues() {
            pTerm.textContent = (kp * error).toFixed(2);
            iTerm.textContent = (ki * integral).toFixed(2);
            dTerm.textContent = (kd * derivative).toFixed(2);
            errorValue.textContent = error.toFixed(2);
            integralValue.textContent = integral.toFixed(2);
            outputValue.textContent = output.toFixed(2);
        }
        
        // 初始重置模拟
        resetSimulation();
    </script>
</body>
</html>
```

