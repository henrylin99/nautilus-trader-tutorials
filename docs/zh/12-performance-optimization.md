# 第12章：性能优化

## 学习目标

- 理解NautilusTrader的性能瓶颈和优化策略
- 掌握内存管理和垃圾回收优化技巧
- 学会使用性能分析工具进行瓶颈定位
- 实现低延迟、高吞吐量的交易系统

## 12.1 性能分析基础

### 12.1.1 性能指标

```python
import cProfile
import pstats
import io
from functools import wraps
from time import perf_counter_ns

# 性能指标收集器
class PerformanceMetrics:
    """性能指标收集器"""

    def __init__(self):
        self.metrics = {
            'latency': [],       # 延迟分布
            'throughput': [],    # 吞吐量
            'memory_usage': [],  # 内存使用
            'cpu_usage': [],     # CPU使用率
            'event_counts': {}   # 事件计数
        }
        self.start_time = None

    def start_recording(self):
        """开始记录"""
        self.start_time = perf_counter_ns()

    def record_latency(self, event_type: str, latency_ns: int):
        """记录延迟"""
        if event_type not in self.metrics['latency']:
            self.metrics['latency'][event_type] = []
        self.metrics['latency'][event_type].append(latency_ns)

    def record_event(self, event_type: str):
        """记录事件"""
        if event_type not in self.metrics['event_counts']:
            self.metrics['event_counts'][event_type] = 0
        self.metrics['event_counts'][event_type] += 1

    def get_percentile(self, latencies: list, percentile: float) -> float:
        """计算百分位数"""
        if not latencies:
            return 0.0
        latencies.sort()
        index = int(len(latencies) * percentile / 100)
        return latencies[min(index, len(latencies) - 1)]

    def generate_report(self) -> dict:
        """生成性能报告"""
        report = {
            'total_runtime_ns': perf_counter_ns() - self.start_time,
            'event_summary': self.metrics['event_counts']
        }

        # 延迟统计
        for event_type, latencies in self.metrics['latency'].items():
            if latencies:
                report[f'{event_type}_latency'] = {
                    'count': len(latencies),
                    'min_ns': min(latencies),
                    'max_ns': max(latencies),
                    'avg_ns': sum(latencies) / len(latencies),
                    'p50_ns': self.get_percentile(latencies, 50),
                    'p90_ns': self.get_percentile(latencies, 90),
                    'p95_ns': self.get_percentile(latencies, 95),
                    'p99_ns': self.get_percentile(latencies, 99)
                }

        return report

# 性能装饰器
def profile_performance(metrics: PerformanceMetrics, event_type: str):
    """性能分析装饰器"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            start = perf_counter_ns()
            result = func(*args, **kwargs)
            end = perf_counter_ns()
            metrics.record_latency(event_type, end - start)
            metrics.record_event(event_type)
            return result
        return wrapper
    return decorator

# 使用示例
from nautilus_trader.adapters.betfair.providers import BetfairInstrumentProvider
from nautilus_trader.model.identifiers import InstrumentId
from nautilus_trader.model.objects import Price

class OptimizedInstrumentProvider(BetfairInstrumentProvider):
    """优化的行情数据提供器"""

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.metrics = PerformanceMetrics()
        self.metrics.start_recording()
        self._price_cache = {}  # 价格缓存
        self._update_queue = []  # 更新队列

    @profile_performance(metrics=None, event_type="quote_tick")
    def process_quote_tick(self, tick):
        """处理行情tick（优化版本）"""
        # 使用缓存减少重复计算
        instrument_id = tick.instrument_id
        if instrument_id not in self._price_cache:
            self._price_cache[instrument_id] = {
                'bid': None,
                'ask': None,
                'last_update': 0
            }

        # 批量处理更新
        self._update_queue.append(tick)
        if len(self._update_queue) >= 100:  # 批量大小
            self._process_batch()

    def _process_batch(self):
        """批量处理更新"""
        start = perf_counter_ns()

        for tick in self._update_queue:
            self._update_price_cache(tick)

        self._update_queue.clear()

        # 记录批处理性能
        processing_time = perf_counter_ns() - start
        self.metrics.record_latency('batch_process', processing_time)
```

### 12.1.2 内存优化

```python
import gc
import psutil
import numpy as np
from typing import List, Dict
from dataclasses import dataclass

# 内存监控器
class MemoryMonitor:
    """内存使用监控器"""

    def __init__(self):
        self.process = psutil.Process()
        self.baseline_memory = self.get_memory_usage()
        self.peak_memory = self.baseline_memory
        self.memory_history = []

    def get_memory_usage(self) -> float:
        """获取当前内存使用量(MB)"""
        return self.process.memory_info().rss / 1024 / 1024

    def record_memory(self):
        """记录内存使用"""
        current = self.get_memory_usage()
        self.peak_memory = max(self.peak_memory, current)
        self.memory_history.append(current)

    def get_memory_stats(self) -> dict:
        """获取内存统计"""
        return {
            'baseline_mb': self.baseline_memory,
            'peak_mb': self.peak_memory,
            'current_mb': self.get_memory_usage(),
            'increase_mb': self.get_memory_usage() - self.baseline_memory,
            'avg_mb': np.mean(self.memory_history) if self.memory_history else 0
        }

# 对象池模式
class ObjectPool:
    """对象池，减少对象创建开销"""

    def __init__(self, create_func, reset_func=None, max_size=1000):
        self.create_func = create_func
        self.reset_func = reset_func
        self.max_size = max_size
        self._pool = []
        self._created_count = 0
        self._reused_count = 0

    def acquire(self):
        """获取对象"""
        if self._pool:
            self._reused_count += 1
            return self._pool.pop()
        else:
            self._created_count += 1
            return self.create_func()

    def release(self, obj):
        """释放对象"""
        if len(self._pool) < self.max_size:
            if self.reset_func:
                self.reset_func(obj)
            self._pool.append(obj)

    def get_stats(self):
        """获取池统计"""
        return {
            'pool_size': len(self._pool),
            'created': self._created_count,
            'reused': self._reused_count,
            'reuse_rate': self._reused_count / (self._created_count + self._reused_count)
        }

# 价格对象池
def create_price():
    """创建价格对象"""
    return Price(0, 0)

def reset_price(price):
    """重置价格对象"""
    price.raw = 0
    price.precision = 0

price_pool = ObjectPool(create_price, reset_price, max_size=10000)

# 使用对象池的价格处理器
class OptimizedPriceProcessor:
    """优化的价格处理器"""

    def __init__(self):
        self.price_pool = price_pool
        self.memory_monitor = MemoryMonitor()

    def process_prices(self, raw_prices: List[float]) -> List[Price]:
        """处理价格列表（使用对象池）"""
        self.memory_monitor.record_memory()

        prices = []
        for raw_price in raw_prices:
            price = self.price_pool.acquire()
            price.raw = int(raw_price * 100000)  # 转换为整数
            price.precision = 5
            prices.append(price)

        # 处理完后释放对象
        # 注意：在实际使用中需要确保不再需要这些对象
        return prices

    def release_prices(self, prices: List[Price]):
        """释放价格对象"""
        for price in prices:
            self.price_pool.release(price)

# 零拷贝数据处理
class ZeroCopyDataHandler:
    """零拷贝数据处理器"""

    def __init__(self):
        self.data_buffer = np.array([], dtype=np.float64)
        self.view_windows = []

    def add_data(self, new_data: np.ndarray):
        """添加数据（使用内存视图）"""
        # 使用视图避免数据拷贝
        if self.data_buffer.size == 0:
            self.data_buffer = new_data
        else:
            # 扩展缓冲区
            new_buffer = np.empty(self.data_buffer.size + new_data.size, dtype=np.float64)
            new_buffer[:self.data_buffer.size] = self.data_buffer
            new_buffer[self.data_buffer.size:] = new_data
            self.data_buffer = new_buffer

    def get_window(self, start: int, end: int) -> np.ndarray:
        """获取数据窗口（返回视图）"""
        return self.data_buffer[start:end]

    def process_sliding_window(self, window_size: int):
        """处理滑动窗口"""
        for i in range(len(self.data_buffer) - window_size + 1):
            window = self.data_buffer[i:i+window_size]
            # 处理窗口数据（零拷贝）
            yield window

# 内存映射数据处理
import mmap
import os

class MemoryMappedDataLoader:
    """内存映射数据加载器"""

    def __init__(self, file_path: str):
        self.file_path = file_path
        self.file_size = os.path.getsize(file_path)
        self.mmap = None
        self.view = None

    def __enter__(self):
        """进入上下文"""
        self.file = open(self.file_path, 'rb')
        self.mmap = mmap.mmap(self.file.fileno(), 0, access=mmap.ACCESS_READ)
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        """退出上下文"""
        if self.mmap:
            self.mmap.close()
        if self.file:
            self.file.close()

    def get_view(self, offset: int = 0, size: int = None) -> memoryview:
        """获取内存视图"""
        if size is None:
            size = self.file_size - offset
        return memoryview(self.mmap)[offset:offset+size]

    def read_struct(self, offset: int, struct_format: str):
        """读取结构化数据"""
        import struct
        size = struct.calcsize(struct_format)
        view = self.get_view(offset, size)
        return struct.unpack(struct_format, view.tobytes())
```

## 12.2 并发优化

### 12.2.1 异步处理

```python
import asyncio
import uvloop
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor
from typing import Callable, Any, List
import time

# 异步任务调度器
class AsyncTaskScheduler:
    """异步任务调度器"""

    def __init__(self, max_workers: int = 4):
        self.executor = ThreadPoolExecutor(max_workers=max_workers)
        self.loop = None
        self.tasks = []
        self.task_queue = asyncio.Queue()
        self.result_queue = asyncio.Queue()

    async def initialize(self):
        """初始化事件循环"""
        # 设置高性能事件循环
        if hasattr(asyncio, 'set_event_loop_policy'):
            asyncio.set_event_loop_policy(uvloop.EventLoopPolicy())
        self.loop = asyncio.get_running_loop()
        # 启动工作协程
        asyncio.create_task(self._worker())

    async def submit_task(self, func: Callable, *args, **kwargs) -> str:
        """提交任务"""
        task_id = f"task_{len(self.tasks)}"
        task = {
            'id': task_id,
            'func': func,
            'args': args,
            'kwargs': kwargs
        }
        await self.task_queue.put(task)
        self.tasks.append(task_id)
        return task_id

    async def _worker(self):
        """工作协程"""
        while True:
            task = await self.task_queue.get()
            try:
                # 在线程池中执行CPU密集型任务
                result = await self.loop.run_in_executor(
                    self.executor,
                    task['func'],
                    *task['args'],
                    **task['kwargs']
                )
                await self.result_queue.put({
                    'id': task['id'],
                    'status': 'completed',
                    'result': result
                })
            except Exception as e:
                await self.result_queue.put({
                    'id': task['id'],
                    'status': 'failed',
                    'error': str(e)
                })
            finally:
                self.task_queue.task_done()

    async def get_result(self, task_id: str, timeout: float = None) -> Any:
        """获取任务结果"""
        start_time = time.time()
        while True:
            if timeout and time.time() - start_time > timeout:
                raise TimeoutError(f"Task {task_id} timed out")

            # 检查结果队列
            try:
                result = self.result_queue.get_nowait()
                if result['id'] == task_id:
                    if result['status'] == 'completed':
                        return result['result']
                    else:
                        raise Exception(result['error'])
                else:
                    # 把结果放回队列
                    await self.result_queue.put(result)
            except asyncio.QueueEmpty:
                pass

            await asyncio.sleep(0.01)  # 避免忙等待

    async def shutdown(self):
        """关闭调度器"""
        await self.task_queue.join()
        self.executor.shutdown(wait=True)

# 批处理器
class AsyncBatchProcessor:
    """异步批处理器"""

    def __init__(self, batch_size: int = 100, timeout: float = 0.1):
        self.batch_size = batch_size
        self.timeout = timeout
        self.buffer = []
        self.last_flush = time.time()
        self.scheduler = AsyncTaskScheduler()
        self.processing_func = None

    async def initialize(self, process_func: Callable):
        """初始化处理器"""
        self.processing_func = process_func
        await self.scheduler.initialize()

    async def add_item(self, item: Any):
        """添加项目到批次"""
        self.buffer.append(item)

        # 检查是否需要处理
        current_time = time.time()
        should_process = (
            len(self.buffer) >= self.batch_size or
            (self.buffer and current_time - self.last_flush > self.timeout)
        )

        if should_process:
            await self._process_batch()

    async def _process_batch(self):
        """处理批次"""
        if not self.buffer:
            return

        batch = self.buffer.copy()
        self.buffer.clear()
        self.last_flush = time.time()

        # 提交批处理任务
        task_id = await self.scheduler.submit_task(
            self.processing_func,
            batch
        )

        # 可以选择等待结果或异步处理
        # await self.scheduler.get_result(task_id)

    async def flush(self):
        """强制处理剩余项目"""
        await self._process_batch()

    async def shutdown(self):
        """关闭处理器"""
        await self.flush()
        await self.scheduler.shutdown()

# 协程池
class CoroutinePool:
    """协程池管理器"""

    def __init__(self, pool_size: int = 10):
        self.pool_size = pool_size
        self.semaphore = asyncio.Semaphore(pool_size)
        self.active_tasks = set()
        self.completed_tasks = 0
        self.failed_tasks = 0

    async def submit(self, coro):
        """提交协程"""
        async def wrapped_coro():
            async with self.semaphore:
                try:
                    result = await coro
                    self.completed_tasks += 1
                    return result
                except Exception as e:
                    self.failed_tasks += 1
                    raise
                finally:
                    self.active_tasks.discard(coro)

        task = asyncio.create_task(wrapped_coro())
        self.active_tasks.add(task)
        return task

    async def wait_for_all(self, timeout: float = None):
        """等待所有任务完成"""
        if self.active_tasks:
            await asyncio.wait_for(
                asyncio.gather(*self.active_tasks, return_exceptions=True),
                timeout=timeout
            )

    def get_stats(self):
        """获取统计信息"""
        return {
            'active_tasks': len(self.active_tasks),
            'completed_tasks': self.completed_tasks,
            'failed_tasks': self.failed_tasks,
            'pool_size': self.pool_size,
            'available_slots': self.semaphore._value
        }

# 使用示例：高性能数据处理管道
class HighPerformanceDataPipeline:
    """高性能数据处理管道"""

    def __init__(self):
        self.stage1_pool = CoroutinePool(pool_size=20)  # 数据接收
        self.stage2_pool = CoroutinePool(pool_size=10)  # 数据处理
        self.stage3_pool = CoroutinePool(pool_size=5)   # 结果输出
        self.batch_processor = AsyncBatchProcessor(batch_size=1000)
        self.metrics = PerformanceMetrics()

    async def initialize(self):
        """初始化管道"""
        await self.batch_processor.initialize(self._process_data_batch)
        self.metrics.start_recording()

    async def process_data_stream(self, data_stream):
        """处理数据流"""
        tasks = []

        async for data_item in data_stream:
            # Stage 1: 快速接收和验证
            task1 = await self.stage1_pool.submit(
                self._validate_data(data_item)
            )

            # Stage 2: 批量处理
            await self.batch_processor.add_item(data_item)

            # Stage 3: 异步输出
            if task1:
                task3 = await self.stage3_pool.submit(
                    self._output_result(task1)
                )
                tasks.append(task3)

        # 等待所有任务完成
        await self.stage1_pool.wait_for_all()
        await self.stage3_pool.wait_for_all()

        # 获取性能报告
        return self.metrics.generate_report()

    async def _validate_data(self, data):
        """验证数据"""
        # 简单的数据验证
        if not data:
            return None
        return data

    async def _output_result(self, result):
        """输出结果"""
        # 输出到数据库或文件
        pass

    def _process_data_batch(self, batch):
        """批量处理数据（CPU密集型）"""
        # 这里是CPU密集型计算
        import numpy as np
        data_array = np.array(batch)
        # 执行计算
        result = np.mean(data_array)
        return result
```

### 12.2.2 多进程优化

```python
import multiprocessing as mp
from multiprocessing import shared_memory
import numpy as np
from typing import Any, List, Tuple
import pickle

# 共享内存管理器
class SharedMemoryManager:
    """共享内存管理器"""

    def __init__(self):
        self.shared_blocks = {}
        self.shapes = {}
        self.dtypes = {}

    def create_array(self, name: str, shape: Tuple, dtype=np.float64):
        """创建共享数组"""
        size = np.prod(shape) * np.dtype(dtype).itemsize

        # 创建共享内存块
        shm = shared_memory.SharedMemory(create=True, size=size, name=name)
        self.shared_blocks[name] = shm
        self.shapes[name] = shape
        self.dtypes[name] = dtype

        # 创建numpy数组视图
        array = np.ndarray(shape, dtype=dtype, buffer=shm.buf)
        return array

    def get_array(self, name: str) -> np.ndarray:
        """获取共享数组"""
        if name not in self.shared_blocks:
            # 尝试连接到已存在的共享内存
            shm = shared_memory.SharedMemory(name=name)
            self.shared_blocks[name] = shm

        shm = self.shared_blocks[name]
        shape = self.shapes.get(name, ())
        dtype = self.dtypes.get(name, np.float64)

        return np.ndarray(shape, dtype=dtype, buffer=shm.buf)

    def cleanup(self, name: str = None):
        """清理共享内存"""
        if name:
            if name in self.shared_blocks:
                self.shared_blocks[name].close()
                self.shared_blocks[name].unlink()
                del self.shared_blocks[name]
        else:
            # 清理所有
            for name in list(self.shared_blocks.keys()):
                self.cleanup(name)

# 多进程数据处理器
def worker_process(data_name: str, result_name: str,
                   start_idx: int, end_idx: int,
                   worker_id: int) -> dict:
    """工作进程函数"""
    shm_manager = SharedMemoryManager()

    # 获取共享数据
    data_array = shm_manager.get_array(data_name)
    result_array = shm_manager.get_array(result_name)

    # 处理数据段
    worker_data = data_array[start_idx:end_idx]

    # 执行计算（示例：计算移动平均）
    window_size = 20
    results = []

    for i in range(len(worker_data) - window_size + 1):
        window = worker_data[i:i+window_size]
        ma = np.mean(window)
        results.append(ma)

    # 写入结果
    result_start = start_idx
    result_end = result_start + len(results)
    result_array[result_start:result_end] = results

    return {
        'worker_id': worker_id,
        'processed': len(results),
        'start_idx': start_idx,
        'end_idx': end_idx
    }

class MultiprocessDataProcessor:
    """多进程数据处理器"""

    def __init__(self, num_processes: int = None):
        self.num_processes = num_processes or mp.cpu_count()
        self.shm_manager = SharedMemoryManager()
        self.process_pool = None

    def process_large_dataset(self, data: np.ndarray) -> np.ndarray:
        """处理大型数据集"""
        data_size = len(data)
        chunk_size = data_size // self.num_processes

        # 创建共享内存
        data_name = "shared_data"
        result_name = "shared_result"

        # 共享输入数据
        shared_data = self.shm_manager.create_array(data_name, data.shape, data.dtype)
        shared_data[:] = data[:]

        # 创建结果数组
        result_shape = data.shape
        shared_result = self.shm_manager.create_array(result_name, result_shape, np.float64)

        # 启动进程池
        with mp.Pool(self.num_processes) as pool:
            tasks = []

            for i in range(self.num_processes):
                start_idx = i * chunk_size
                end_idx = start_idx + chunk_size if i < self.num_processes - 1 else data_size

                task = pool.apply_async(
                    worker_process,
                    args=(data_name, result_name, start_idx, end_idx, i)
                )
                tasks.append(task)

            # 收集结果
            results = [task.get() for task in tasks]

        # 获取结果数组
        result = self.shm_manager.get_array(result_name).copy()

        # 清理共享内存
        self.shm_manager.cleanup()

        return result

    def __del__(self):
        """析构函数"""
        if hasattr(self, 'shm_manager'):
            self.shm_manager.cleanup()

# 进程间通信优化
class OptimizedIPC:
    """优化的进程间通信"""

    def __init__(self):
        self.read_pipe, self.write_pipe = mp.Pipe(duplex=False)
        self.shared_buffer = None

    def create_shared_buffer(self, size: int):
        """创建共享缓冲区"""
        self.shared_buffer = mp.Array('b', size)

    def send_large_data(self, data: Any):
        """发送大数据"""
        # 序列化数据
        serialized = pickle.dumps(data)

        # 检查是否需要使用共享缓冲区
        if len(serialized) > 1024 * 1024:  # 1MB
            if not self.shared_buffer or len(self.shared_buffer) < len(serialized):
                self.create_shared_buffer(len(serialized))

            # 写入共享缓冲区
            buffer_view = memoryview(self.shared_buffer).cast('B')
            buffer_view[:len(serialized)] = serialized

            # 发送元数据
            self.write_pipe.send({
                'type': 'shared_buffer',
                'size': len(serialized)
            })
        else:
            # 直接发送
            self.write_pipe.send({
                'type': 'direct',
                'data': serialized
            })

    def receive_large_data(self) -> Any:
        """接收大数据"""
        metadata = self.read_pipe.recv()

        if metadata['type'] == 'shared_buffer':
            # 从共享缓冲区读取
            buffer_view = memoryview(self.shared_buffer).cast('B')
            serialized = bytes(buffer_view[:metadata['size']])
        else:
            # 直接数据
            serialized = metadata['data']

        # 反序列化
        return pickle.loads(serialized)

# 并行计算框架
class ParallelComputeFramework:
    """并行计算框架"""

    def __init__(self, num_workers: int = None):
        self.num_workers = num_workers or mp.cpu_count()
        self.workers = []
        self.task_queue = mp.Queue()
        self.result_queue = mp.Queue()
        self.shm_manager = SharedMemoryManager()

    def initialize(self):
        """初始化工作进程"""
        for i in range(self.num_workers):
            worker = mp.Process(
                target=self._worker_loop,
                args=(i,)
            )
            worker.start()
            self.workers.append(worker)

    def _worker_loop(self, worker_id: int):
        """工作进程循环"""
        while True:
            try:
                task = self.task_queue.get()
                if task is None:  # 停止信号
                    break

                # 执行任务
                func_name = task['func']
                args = task.get('args', [])
                kwargs = task.get('kwargs', {})

                # 动态导入函数
                module_name, func_name = func_name.rsplit('.', 1)
                module = __import__(module_name, fromlist=[func_name])
                func = getattr(module, func_name)

                result = func(*args, **kwargs)

                # 发送结果
                self.result_queue.put({
                    'worker_id': worker_id,
                    'task_id': task.get('task_id'),
                    'result': result
                })

            except Exception as e:
                self.result_queue.put({
                    'worker_id': worker_id,
                    'task_id': task.get('task_id'),
                    'error': str(e)
                })

    def submit_task(self, func: str, *args, **kwargs) -> str:
        """提交任务"""
        import uuid
        task_id = str(uuid.uuid4())

        task = {
            'task_id': task_id,
            'func': func,
            'args': args,
            'kwargs': kwargs
        }

        self.task_queue.put(task)
        return task_id

    def get_result(self, timeout: float = None):
        """获取结果"""
        return self.result_queue.get(timeout=timeout)

    def shutdown(self):
        """关闭框架"""
        # 发送停止信号
        for _ in range(self.num_workers):
            self.task_queue.put(None)

        # 等待工作进程结束
        for worker in self.workers:
            worker.join()

        # 清理共享内存
        self.shm_manager.cleanup()
```

## 12.3 高频交易优化

### 12.3.1 微秒级延迟优化

```python
import time
import ctypes
import numpy as np
from numba import jit, cuda
import cython

# 使用Cython优化关键路径
# 编译命令: cythonize -i fast_order.pyx

"""
# fast_order.pyx
cimport cython
cimport numpy as np
from libc.time cimport clock_gettime, CLOCK_REALTIME

@cython.boundscheck(False)
@cython.wraparound(False)
@cython.cdivision(True)
def fast_calculate_price(double[:] prices, int period):
    cdef int i
    cdef double sum_price = 0.0
    cdef int n = prices.shape[0]

    for i in range(period):
        sum_price += prices[i]

    return sum_price / period

@cython.boundscheck(False)
cpdef uint64_t get_timestamp_ns():
    cimport libc.time
    cdef libc.time.timespec ts
    clock_gettime(CLOCK_REALTIME, &ts)
    return (<uint64_t>ts.tv_sec) * 1000000000 + (<uint64_t>ts.tv_nsec)
"""

# FPGA加速的数据结构
class FPGAAlignedBuffer:
    """FPGA对齐的缓冲区"""

    def __init__(self, size: int, alignment: int = 64):
        self.size = size
        self.alignment = alignment
        self.buffer = self._allocate_aligned_memory(size, alignment)

    def _allocate_aligned_memory(self, size: int, alignment: int):
        """分配对齐内存"""
        # 使用ctypes分配对齐内存
        buffer = (ctypes.c_ubyte * size)()
        address = ctypes.addressof(buffer)

        # 计算对齐地址
        aligned_address = (address + alignment - 1) & ~(alignment - 1)

        # 创建对齐的内存视图
        aligned_size = size - (aligned_address - address)
        aligned_buffer = (ctypes.c_ubyte * aligned_size).from_address(aligned_address)

        return aligned_buffer

    def write_atomic(self, offset: int, value: int, size: int = 8):
        """原子写入"""
        # 这里应该使用原子操作
        for i in range(size):
            self.buffer[offset + i] = (value >> (i * 8)) & 0xFF

    def read_atomic(self, offset: int, size: int = 8) -> int:
        """原子读取"""
        value = 0
        for i in range(size):
            value |= self.buffer[offset + i] << (i * 8)
        return value

# 高频交易数据结构
@jit(nopython=True, cache=True)
def fast_order_matching(
    bid_prices: np.ndarray,
    bid_sizes: np.ndarray,
    ask_prices: np.ndarray,
    ask_sizes: np.ndarray,
    order_price: float,
    order_size: float,
    is_buy: bool
) -> tuple:
    """快速订单匹配"""

    executed_size = 0.0
    avg_price = 0.0
    total_cost = 0.0

    if is_buy:  # 买单，匹配卖盘
        for i in range(len(ask_prices)):
            if executed_size >= order_size:
                break

            if ask_prices[i] <= order_price:
                fill_size = min(ask_sizes[i], order_size - executed_size)
                executed_size += fill_size
                total_cost += fill_size * ask_prices[i]

    else:  # 卖单，匹配买盘
        for i in range(len(bid_prices)):
            if executed_size >= order_size:
                break

            if bid_prices[i] >= order_price:
                fill_size = min(bid_sizes[i], order_size - executed_size)
                executed_size += fill_size
                total_cost += fill_size * bid_prices[i]

    if executed_size > 0:
        avg_price = total_cost / executed_size

    return executed_size, avg_price

# GPU加速的风险检查
@cuda.jit
def gpu_risk_check(
    positions: np.ndarray,
    prices: np.ndarray,
    limits: np.ndarray,
    violations: np.ndarray
):
    """GPU并行的风险检查"""

    idx = cuda.grid(1)
    if idx < positions.shape[0]:
        # 计算当前盈亏
        pnl = positions[idx] * prices[idx]

        # 检查是否超过限制
        if abs(pnl) > limits[idx]:
            violations[idx] = 1
        else:
            violations[idx] = 0

class HFTOptimizer:
    """高频交易优化器"""

    def __init__(self):
        self.tick_processor = None
        self.order_book = None
        self.risk_engine = None
        self.execution_engine = None

    def optimize_tick_to_trade(self):
        """优化tick-to-trade延迟"""

        # 1. CPU亲和性设置
        import os
        os.sched_setaffinity(0, {0})  # 绑定到CPU 0

        # 2. 实时优先级
        import psutil
        p = psutil.Process()
        if hasattr(p, 'ionice'):
            p.ionice(psutil.IOPRIO_HIGHEST)  # 最高IO优先度
        if hasattr(p, 'nice'):
            p.nice(-20)  # 最高CPU优先级

        # 3. 预热缓存
        self._warmup_caches()

        # 4. 禁用交换
        self._disable_swap()

        # 5. 优化网络
        self._optimize_network()

    def _warmup_caches(self):
        """预热CPU缓存"""
        # 预分配所有需要的内存
        large_arrays = []
        for _ in range(10):
            arr = np.zeros(1000000, dtype=np.float64)
            arr.fill(1.0)
            large_arrays.append(arr)

        # 触发缓存填充
        for arr in large_arrays:
            _ = np.sum(arr)

    def _disable_swap(self):
        """禁用交换"""
        import subprocess
        try:
            subprocess.run(['sudo', 'swapoff', '-a'], check=True)
        except:
            pass  # 可能需要root权限

    def _optimize_network(self):
        """优化网络设置"""
        import subprocess
        try:
            # 设置网络缓冲区
            subprocess.run([
                'sudo', 'sysctl', '-w', 'net.core.rmem_max=134217728'
            ], check=True)
            subprocess.run([
                'sudo', 'sysctl', '-w', 'net.core.wmem_max=134217728'
            ], check=True)
        except:
            pass

    def measure_latency(self, iterations: int = 10000) -> dict:
        """测量延迟"""
        latencies = []

        for _ in range(iterations):
            start = time.perf_counter_ns()

            # 模拟tick-to-trade操作
            self._simulate_tick_processing()

            end = time.perf_counter_ns()
            latencies.append(end - start)

        latencies = np.array(latencies)

        return {
            'min_ns': np.min(latencies),
            'max_ns': np.max(latencies),
            'mean_ns': np.mean(latencies),
            'p50_ns': np.percentile(latencies, 50),
            'p90_ns': np.percentile(latencies, 90),
            'p95_ns': np.percentile(latencies, 95),
            'p99_ns': np.percentile(latencies, 99),
            'throughput_millions_per_sec': iterations / (np.sum(latencies) / 1e9) / 1e6
        }

    def _simulate_tick_processing(self):
        """模拟tick处理"""
        # 模拟数据处理
        data = np.random.randn(1000)
        result = np.mean(data)

        # 模拟订单处理
        order_book = np.random.rand(1000, 2)
        executed = fast_order_matching(
            order_book[:, 0], order_book[:, 1],
            order_book[:, 0], order_book[:, 1],
            result, 100.0, True
        )

# 低延迟网络通信
import socket
import struct

class LowLatencyNetwork:
    """低延迟网络通信"""

    def __init__(self, host: str, port: int):
        self.host = host
        self.port = port
        self.socket = None
        self.buffer_size = 65536  # 64KB

    def connect(self):
        """建立连接"""
        # 创建TCP套接字
        self.socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

        # 设置套接字选项
        self.socket.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)
        self.socket.setsockopt(socket.SOL_SOCKET, socket.SO_SNDBUF, self.buffer_size)
        self.socket.setsockopt(socket.SOL_SOCKET, socket.SO_RCVBUF, self.buffer_size)

        # 连接
        self.socket.connect((self.host, self.port))

    def send_market_data(self, data: dict):
        """发送市场数据"""
        # 自定义二进制协议
        message = self._encode_market_data(data)
        self.socket.sendall(message)

    def _encode_market_data(self, data: dict) -> bytes:
        """编码市场数据"""
        # 简单的二进制编码
        fmt = '!QdI'  # timestamp(8) + price(8) + size(4)
        return struct.pack(fmt,
                          int(data['timestamp'] * 1e9),
                          data['price'],
                          int(data['size']))

    def close(self):
        """关闭连接"""
        if self.socket:
            self.socket.close()
```

## 12.4 实时性能监控

### 12.4.1 系统监控

```python
import psutil
import GPUtil
import nvidia_ml_py3 as nvml
from prometheus_client import CollectorRegistry, Gauge, start_http_server
import time

class SystemMonitor:
    """系统资源监控器"""

    def __init__(self):
        self.registry = CollectorRegistry()
        self.setup_metrics()
        self.monitoring = False

    def setup_metrics(self):
        """设置监控指标"""
        # CPU指标
        self.cpu_usage = Gauge(
            'cpu_usage_percent',
            'CPU使用率',
            registry=self.registry
        )
        self.cpu_temp = Gauge(
            'cpu_temperature_celsius',
            'CPU温度',
            registry=self.registry
        )

        # 内存指标
        self.memory_usage = Gauge(
            'memory_usage_percent',
            '内存使用率',
            registry=self.registry
        )
        self.memory_available = Gauge(
            'memory_available_bytes',
            '可用内存字节数',
            registry=self.registry
        )

        # 网络指标
        self.network_bytes_sent = Gauge(
            'network_bytes_sent_total',
            '网络发送字节数',
            registry=self.registry
        )
        self.network_bytes_recv = Gauge(
            'network_bytes_recv_total',
            '网络接收字节数',
            registry=self.registry
        )

        # GPU指标
        self.gpu_usage = Gauge(
            'gpu_usage_percent',
            'GPU使用率',
            labelnames=['gpu_id'],
            registry=self.registry
        )
        self.gpu_memory_usage = Gauge(
            'gpu_memory_usage_percent',
            'GPU内存使用率',
            labelnames=['gpu_id'],
            registry=self.registry
        )

    def start_monitoring(self, port: int = 8000):
        """启动监控"""
        # 启动Prometheus HTTP服务器
        start_http_server(port, registry=self.registry)
        self.monitoring = True

        # 初始化计数器
        self.initial_net_io = psutil.net_io_counters()

        # 启动监控循环
        import threading
        self.monitor_thread = threading.Thread(target=self._monitor_loop)
        self.monitor_thread.daemon = True
        self.monitor_thread.start()

    def _monitor_loop(self):
        """监控循环"""
        while self.monitoring:
            # 更新CPU指标
            self.cpu_usage.set(psutil.cpu_percent())

            # CPU温度（需要lm-sensors）
            try:
                temps = psutil.sensors_temperatures()
                if 'coretemp' in temps:
                    self.cpu_temp.set(temps['coretemp'][0].current)
            except:
                pass

            # 更新内存指标
            memory = psutil.virtual_memory()
            self.memory_usage.set(memory.percent)
            self.memory_available.set(memory.available)

            # 更新网络指标
            net_io = psutil.net_io_counters()
            self.network_bytes_sent.set(net_io.bytes_sent)
            self.network_bytes_recv.set(net_io.bytes_recv)

            # 更新GPU指标
            try:
                gpus = GPUtil.getGPUs()
                for gpu in gpus:
                    self.gpu_usage.labels(gpu_id=gpu.id).set(gpu.load * 100)
                    self.gpu_memory_usage.labels(gpu_id=gpu.id).set(
                        gpu.memoryUtil * 100
                    )
            except:
                pass

            time.sleep(1)  # 每秒更新

    def stop_monitoring(self):
        """停止监控"""
        self.monitoring = False

# 应用性能监控
class ApplicationMonitor:
    """应用性能监控器"""

    def __init__(self):
        self.metrics = {}
        self.timers = {}
        self.counters = {}
        self.histograms = {}

    def timer(self, name: str):
        """计时器装饰器"""
        def decorator(func):
            def wrapper(*args, **kwargs):
                start_time = time.perf_counter()
                result = func(*args, **kwargs)
                duration = time.perf_counter() - start_time

                # 记录计时
                if name not in self.metrics:
                    self.metrics[name] = []
                self.metrics[name].append(duration)

                return result
            return wrapper
        return decorator

    def counter(self, name: str):
        """计数器装饰器"""
        def decorator(func):
            def wrapper(*args, **kwargs):
                result = func(*args, **kwargs)

                # 增加计数
                if name not in self.counters:
                    self.counters[name] = 0
                self.counters[name] += 1

                return result
            return wrapper
        return decorator

    def histogram(self, name: str, value: float):
        """记录直方图数据"""
        if name not in self.histograms:
            self.histograms[name] = []
        self.histograms[name].append(value)

    def get_percentiles(self, name: str) -> dict:
        """获取百分位数"""
        if name not in self.metrics:
            return {}

        values = sorted(self.metrics[name])
        return {
            'min': min(values),
            'max': max(values),
            'mean': sum(values) / len(values),
            'p50': values[int(len(values) * 0.5)],
            'p90': values[int(len(values) * 0.9)],
            'p95': values[int(len(values) * 0.95)],
            'p99': values[int(len(values) * 0.99)]
        }

    def generate_report(self) -> dict:
        """生成监控报告"""
        report = {
            'counters': self.counters,
            'timers': {}
        }

        for name in self.metrics:
            report['timers'][name] = self.get_percentiles(name)

        return report

# 实时延迟监控
class LatencyMonitor:
    """实时延迟监控器"""

    def __init__(self, window_size: int = 1000):
        self.window_size = window_size
        self.latencies = []
        self.anomalies = []
        self.baseline_latency = None
        self.threshold_multiplier = 3.0

    def record_latency(self, latency_ns: int):
        """记录延迟"""
        self.latencies.append(latency_ns)

        # 保持窗口大小
        if len(self.latencies) > self.window_size:
            self.latencies.pop(0)

        # 检测异常
        self._detect_anomaly(latency_ns)

    def _detect_anomaly(self, latency_ns: int):
        """检测延迟异常"""
        if len(self.latencies) < 100:  # 需要足够的数据
            return

        if not self.baseline_latency:
            # 计算基线延迟（中位数）
            self.baseline_latency = np.median(self.latencies[:-10])

        # 检查是否超过阈值
        threshold = self.baseline_latency * self.threshold_multiplier
        if latency_ns > threshold:
            self.anomalies.append({
                'timestamp': time.time(),
                'latency_ns': latency_ns,
                'baseline_ns': self.baseline_latency,
                'threshold_ns': threshold
            })

    def get_real_time_stats(self) -> dict:
        """获取实时统计"""
        if not self.latencies:
            return {}

        recent_latencies = self.latencies[-100:]  # 最近100个

        return {
            'current_ns': self.latencies[-1],
            'avg_recent_ns': np.mean(recent_latencies),
            'p99_recent_ns': np.percentile(recent_latencies, 99),
            'anomaly_count': len(self.anomalies),
            'anomaly_rate': len(self.anomalies) / len(self.latencies) * 100
        }

# 性能告警系统
class PerformanceAlertSystem:
    """性能告警系统"""

    def __init__(self):
        self.alerts = []
        self.rules = []
        self.notification_handlers = []

    def add_rule(self, name: str, metric: str, condition: str, threshold: float):
        """添加告警规则"""
        self.rules.append({
            'name': name,
            'metric': metric,
            'condition': condition,
            'threshold': threshold,
            'last_triggered': 0
        })

    def check_metrics(self, metrics: dict):
        """检查指标"""
        current_time = time.time()

        for rule in self.rules:
            metric_value = metrics.get(rule['metric'], 0)

            # 评估条件
            triggered = False
            if rule['condition'] == '>':
                triggered = metric_value > rule['threshold']
            elif rule['condition'] == '<':
                triggered = metric_value < rule['threshold']
            elif rule['condition'] == '==':
                triggered = metric_value == rule['threshold']

            # 触发告警
            if triggered and (current_time - rule['last_triggered'] > 60):  # 1分钟冷却
                alert = {
                    'rule': rule['name'],
                    'metric': rule['metric'],
                    'value': metric_value,
                    'threshold': rule['threshold'],
                    'timestamp': current_time
                }

                self.alerts.append(alert)
                rule['last_triggered'] = current_time

                # 发送通知
                self._send_notification(alert)

    def _send_notification(self, alert: dict):
        """发送通知"""
        for handler in self.notification_handlers:
            handler(alert)

    def add_email_handler(self, smtp_server: str, from_email: str, to_emails: list):
        """添加邮件通知处理器"""
        import smtplib
        from email.mime.text import MIMEText

        def email_handler(alert):
            msg = MIMEText(f"""
            性能告警触发！

            规则: {alert['rule']}
            指标: {alert['metric']}
            当前值: {alert['value']}
            阈值: {alert['threshold']}
            时间: {time.ctime(alert['timestamp'])}
            """)

            msg['Subject'] = f"性能告警: {alert['rule']}"
            msg['From'] = from_email
            msg['To'] = ', '.join(to_emails)

            server = smtplib.SMTP(smtp_server)
            server.send_message(msg)
            server.quit()

        self.notification_handlers.append(email_handler)

    def add_slack_handler(self, webhook_url: str):
        """添加Slack通知处理器"""
        import requests

        def slack_handler(alert):
            payload = {
                "text": f"""
                :warning: 性能告警触发！
                规则: {alert['rule']}
                指标: {alert['metric']}
                当前值: {alert['value']}
                阈值: {alert['threshold']}
                """
            }

            requests.post(webhook_url, json=payload)

        self.notification_handlers.append(slack_handler)
```

## 12.5 性能优化最佳实践

### 12.5.1 代码优化清单

```python
# 性能优化清单
class PerformanceOptimizationChecklist:
    """性能优化清单"""

    @staticmethod
    def memory_optimizations():
        """内存优化检查项"""
        return {
            "对象池": "使用对象池减少GC压力",
            "零拷贝": "使用内存视图避免数据拷贝",
            "数据类型": "使用适当的数据类型（float32而非float64）",
            "预分配": "预分配数组大小",
            "惰性计算": "只在需要时计算",
            "内存映射": "大文件使用内存映射",
            "循环引用": "避免循环引用"
        }

    @staticmethod
    def cpu_optimizations():
        """CPU优化检查项"""
        return {
            "向量化": "使用numpy向量化操作",
            "JIT编译": "使用numba JIT编译热路径",
            "Cython": "关键代码用Cython重写",
            "算法复杂度": "优化算法复杂度",
            "缓存友好": "优化数据访问模式",
            "分支预测": "减少条件分支",
            "SIMD": "使用SIMD指令"
        }

    @staticmethod
    def io_optimizations():
        """IO优化检查项"""
        return {
            "批量处理": "批量处理减少IO次数",
            "异步IO": "使用异步IO模型",
            "压缩": "压缩减少传输量",
            "连接池": "复用连接",
            "缓冲": "适当大小的缓冲区",
            "序列化": "高效的序列化格式",
            "预取": "预取数据"
        }

# 自动性能优化工具
class AutoPerformanceOptimizer:
    """自动性能优化工具"""

    def __init__(self, code_path: str):
        self.code_path = code_path
        self.optimizations = []

    def analyze_code(self) -> dict:
        """分析代码找出优化点"""
        import ast
        import os

        issues = []

        # 遍历Python文件
        for root, dirs, files in os.walk(self.code_path):
            for file in files:
                if file.endswith('.py'):
                    file_path = os.path.join(root, file)
                    with open(file_path, 'r') as f:
                        content = f.read()

                    # 解析AST
                    try:
                        tree = ast.parse(content)
                        issues.extend(self._analyze_ast(tree, file_path))
                    except:
                        pass

        return {
            'issues': issues,
            'recommendations': self._generate_recommendations(issues)
        }

    def _analyze_ast(self, tree, file_path: str) -> list:
        """分析AST找出性能问题"""
        issues = []

        for node in ast.walk(tree):
            # 检查循环中的append
            if isinstance(node, ast.For):
                for inner_node in ast.walk(node):
                    if isinstance(inner_node, ast.Call):
                        if (isinstance(inner_node.func, ast.Attribute) and
                            inner_node.func.attr == 'append'):
                            issues.append({
                                'type': 'list_append_in_loop',
                                'file': file_path,
                                'line': node.lineno,
                                'message': '循环中的list.append可以预分配'
                            })

            # 检查重复计算
            if isinstance(node, ast.Call):
                # 这里可以添加更复杂的分析逻辑
                pass

        return issues

    def _generate_recommendations(self, issues: list) -> list:
        """生成优化建议"""
        recommendations = []

        # 按问题类型分组
        issue_types = {}
        for issue in issues:
            issue_type = issue['type']
            if issue_type not in issue_types:
                issue_types[issue_type] = []
            issue_types[issue_type].append(issue)

        # 生成建议
        for issue_type, type_issues in issue_types.items():
            if issue_type == 'list_append_in_loop':
                recommendations.append({
                    'type': 'list_append_in_loop',
                    'count': len(type_issues),
                    'suggestion': '使用numpy数组或预分配列表大小',
                    'example': '''
# 优化前
result = []
for x in data:
    result.append(x * 2)

# 优化后
result = [0] * len(data)
for i, x in enumerate(data):
    result[i] = x * 2

# 或使用numpy
result = np.array(data) * 2
                    '''
                })

        return recommendations

    def apply_optimizations(self, recommendations: list):
        """应用优化建议"""
        # 这里可以实现自动应用优化
        pass

# 性能基准测试套件
class PerformanceBenchmarkSuite:
    """性能基准测试套件"""

    def __init__(self):
        self.benchmarks = {}
        self.results = {}

    def benchmark(self, name: str):
        """基准测试装饰器"""
        def decorator(func):
            def wrapper(*args, **kwargs):
                # 预热
                for _ in range(10):
                    func(*args, **kwargs)

                # 实际测试
                times = []
                for _ in range(100):
                    start = time.perf_counter_ns()
                    result = func(*args, **kwargs)
                    end = time.perf_counter_ns()
                    times.append(end - start)

                # 记录结果
                self.results[name] = {
                    'min_ns': min(times),
                    'max_ns': max(times),
                    'mean_ns': sum(times) / len(times),
                    'std_ns': np.std(times)
                }

                return result
            return wrapper
        self.benchmarks[name] = decorator
        return decorator

    def run_all(self):
        """运行所有基准测试"""
        print("基准测试结果:")
        print("-" * 50)

        for name, result in self.results.items():
            print(f"{name}:")
            print(f"  平均: {result['mean_ns']/1000:.2f} μs")
            print(f"  最小: {result['min_ns']/1000:.2f} μs")
            print(f"  最大: {result['max_ns']/1000:.2f} μs")
            print(f"  标准差: {result['std_ns']/1000:.2f} μs")
            print()

# 使用示例
benchmark = PerformanceBenchmarkSuite()

@benchmark.benchmark("list_creation")
def test_list_creation():
    return [i for i in range(1000)]

@benchmark.benchmark("numpy_array_creation")
def test_numpy_creation():
    return np.arange(1000)

# 运行基准测试
test_list_creation()
test_numpy_creation()
benchmark.run_all()
```

## 12.6 实战案例

### 12.6.1 高频做市策略优化

```python
import numpy as np
from numba import jit, types
from numba.typed import Dict

# 高频做市策略核心
@jit(nopython=True, cache=True, parallel=True)
def高频做市计算(
    mid_prices: np.ndarray,
    spreads: np.ndarray,
    inventory: np.ndarray,
    risk_limits: np.ndarray,
    depth: int
) -> tuple:
    """
    高频做市策略核心计算

    Returns:
        (bid_prices, ask_prices, bid_sizes, ask_sizes)
    """
    n = len(mid_prices)
    bid_prices = np.zeros(n)
    ask_prices = np.zeros(n)
    bid_sizes = np.zeros(n)
    ask_sizes = np.zeros(n)

    # 并行处理每个时间点
    for i in range(n):
        # 当前库存和风险限制
        current_inventory = inventory[i]
        risk_limit = risk_limits[i]

        # 基础价差
        base_spread = spreads[i]

        # 根据库存调整报价
        inventory_skew = current_inventory / risk_limit

        # 计算买卖价格
        bid_price = mid_prices[i] - base_spread * (1 + inventory_skew)
        ask_price = mid_prices[i] + base_spread * (1 - inventory_skew)

        # 根据深度调整报价
        if depth > 1:
            # 多层报价
            for level in range(depth):
                level_spread = base_spread * (1 + level * 0.1)
                bid_prices[i * depth + level] = bid_price - level * level_spread
                ask_prices[i * depth + level] = ask_price + level * level_spread

                # 递减的订单大小
                size = 100 / (level + 1)
                bid_sizes[i * depth + level] = size
                ask_sizes[i * depth + level] = size
        else:
            # 单层报价
            bid_prices[i] = bid_price
            ask_prices[i] = ask_price
            bid_sizes[i] = 100
            ask_sizes[i] = 100

    return bid_prices, ask_prices, bid_sizes, ask_sizes

# 优化的订单簿管理
class OptimizedOrderBook:
    """优化的订单簿管理器"""

    def __init__(self, max_levels: int = 100):
        self.max_levels = max_levels
        self.bid_prices = np.zeros(max_levels, dtype=np.float64)
        self.bid_sizes = np.zeros(max_levels, dtype=np.float64)
        self.ask_prices = np.zeros(max_levels, dtype=np.float64)
        self.ask_sizes = np.zeros(max_levels, dtype=np.float64)
        self.bid_count = 0
        self.ask_count = 0
        self.last_update = 0

    @jit(nopython=True)
    def update_bid(self, price: float, size: float, level: int):
        """更新买盘（JIT优化）"""
        if level < self.max_levels:
            self.bid_prices[level] = price
            self.bid_sizes[level] = size
            self.bid_count = max(self.bid_count, level + 1)
            self.last_update = time.time_ns()

    @jit(nopython=True)
    def update_ask(self, price: float, size: float, level: int):
        """更新卖盘（JIT优化）"""
        if level < self.max_levels:
            self.ask_prices[level] = price
            self.ask_sizes[level] = size
            self.ask_count = max(self.ask_count, level + 1)
            self.last_update = time.time_ns()

    @jit(nopython=True)
    def get_best_bid(self) -> tuple:
        """获取最优买价"""
        if self.bid_count > 0:
            # 价格从高到低排序
            best_idx = 0
            best_price = self.bid_prices[0]
            for i in range(1, self.bid_count):
                if self.bid_prices[i] > best_price and self.bid_sizes[i] > 0:
                    best_price = self.bid_prices[i]
                    best_idx = i
            return best_price, self.bid_sizes[best_idx]
        return 0.0, 0.0

    @jit(nopython=True)
    def get_best_ask(self) -> tuple:
        """获取最优卖价"""
        if self.ask_count > 0:
            # 价格从低到高排序
            best_idx = 0
            best_price = self.ask_prices[0]
            for i in range(1, self.ask_count):
                if self.ask_prices[i] < best_price and self.ask_sizes[i] > 0:
                    best_price = self.ask_prices[i]
                    best_idx = i
            return best_price, self.ask_sizes[best_idx]
        return 0.0, 0.0

# 超低延迟执行引擎
class UltraLowLatencyEngine:
    """超低延迟执行引擎"""

    def __init__(self):
        self.order_book = OptimizedOrderBook()
        self.position_manager = FastPositionManager()
        self.risk_checker = JITRiskChecker()
        self.execution_stats = {
            'orders_sent': 0,
            'orders_filled': 0,
            'avg_latency_ns': 0,
            'min_latency_ns': float('inf'),
            'max_latency_ns': 0
        }

    @jit(nopython=True)
    def process_market_data(self, bid_price: float, ask_price: float,
                           timestamp: int) -> tuple:
        """处理市场数据（JIT优化）"""
        # 更新订单簿
        self.order_book.update_bid(bid_price, 100.0, 0)
        self.order_book.update_ask(ask_price, 100.0, 0)

        # 计算中价
        mid_price = (bid_price + ask_price) / 2.0
        spread = ask_price - bid_price

        # 获取当前持仓
        current_position = self.position_manager.get_position()

        # 风险检查
        risk_ok = self.risk_checker.check_position_limit(current_position)

        # 决定报价
        if risk_ok:
            # 使用JIT优化的做市计算
            bid_prices, ask_prices, bid_sizes, ask_sizes = 高频做市计算(
                np.array([mid_price]),
                np.array([spread / 2.0]),
                np.array([current_position]),
                np.array([1000.0]),  # 风险限制
                1  # 深度
            )

            return (
                bid_prices[0], ask_prices[0],
                bid_sizes[0], ask_sizes[0]
            )

        return 0.0, 0.0, 0.0, 0.0

    def execute_order(self, side: str, price: float, size: float) -> bool:
        """执行订单"""
        start_time = time.perf_counter_ns()

        # 快速风险检查
        if not self.risk_checker.fast_risk_check(side, size):
            return False

        # 模拟订单发送
        success = self._send_order(side, price, size)

        # 更新统计
        end_time = time.perf_counter_ns()
        latency = end_time - start_time

        self.execution_stats['orders_sent'] += 1
        if success:
            self.execution_stats['orders_filled'] += 1
            self.position_manager.update_position(side, size)

        # 更新延迟统计
        self._update_latency_stats(latency)

        return success

    def _send_order(self, side: str, price: float, size: float) -> bool:
        """发送订单（模拟）"""
        # 这里应该是实际的订单发送逻辑
        return True

    def _update_latency_stats(self, latency_ns: int):
        """更新延迟统计"""
        stats = self.execution_stats
        stats['min_latency_ns'] = min(stats['min_latency_ns'], latency_ns)
        stats['max_latency_ns'] = max(stats['max_latency_ns'], latency_ns)

        # 计算移动平均
        n = stats['orders_sent']
        stats['avg_latency_ns'] = (
            (stats['avg_latency_ns'] * (n - 1) + latency_ns) / n
        )

# 快速持仓管理器
@jit(nopython=True)
class FastPositionManager:
    """快速持仓管理器"""

    def __init__(self):
        self.positions = Dict.empty(key_type=types.unicode_type, value_type=types.float64)
        self.total_position = 0.0

    def update_position(self, symbol: str, quantity: float):
        """更新持仓"""
        if symbol in self.positions:
            self.positions[symbol] += quantity
        else:
            self.positions[symbol] = quantity
        self.total_position += quantity

    def get_position(self, symbol: str = None) -> float:
        """获取持仓"""
        if symbol:
            return self.positions.get(symbol, 0.0)
        return self.total_position

# JIT风险检查器
@jit(nopython=True)
class JITRiskChecker:
    """JIT优化的风险检查器"""

    def __init__(self):
        self.max_position = 1000.0
        self.max_order_size = 100.0
        self.max_notional = 100000.0

    def check_position_limit(self, position: float) -> bool:
        """检查持仓限制"""
        return abs(position) < self.max_position

    def fast_risk_check(self, side: str, size: float) -> bool:
        """快速风险检查"""
        return size < self.max_order_size

# 性能测试框架
class HFTPerformanceTest:
    """高频交易性能测试"""

    def __init__(self):
        self.engine = UltraLowLatencyEngine()
        self.test_data = self._generate_test_data()

    def _generate_test_data(self, n: int = 100000) -> list:
        """生成测试数据"""
        np.random.seed(42)
        data = []
        base_price = 100.0

        for i in range(n):
            # 随机游走价格
            base_price += np.random.randn() * 0.01

            # 买卖价差
            spread = np.random.uniform(0.01, 0.05)

            data.append({
                'bid': base_price - spread / 2,
                'ask': base_price + spread / 2,
                'timestamp': i
            })

        return data

    def run_latency_test(self) -> dict:
        """运行延迟测试"""
        latencies = []

        for tick in self.test_data:
            start = time.perf_counter_ns()

            # 处理tick
            bid, ask, bid_size, ask_size = self.engine.process_market_data(
                tick['bid'], tick['ask'], tick['timestamp']
            )

            # 执行订单
            if bid > 0:
                self.engine.execute_order('BUY', bid, bid_size)
            if ask > 0:
                self.engine.execute_order('SELL', ask, ask_size)

            end = time.perf_counter_ns()
            latencies.append(end - start)

        # 统计结果
        latencies = np.array(latencies)

        return {
            'total_ticks': len(self.test_data),
            'avg_latency_ns': np.mean(latencies),
            'p50_latency_ns': np.percentile(latencies, 50),
            'p90_latency_ns': np.percentile(latencies, 90),
            'p99_latency_ns': np.percentile(latencies, 99),
            'max_latency_ns': np.max(latencies),
            'min_latency_ns': np.min(latencies),
            'throughput_millions_per_sec': len(latencies) / (np.sum(latencies) / 1e9) / 1e6
        }

    def run_memory_test(self) -> dict:
        """运行内存测试"""
        import tracemalloc

        tracemalloc.start()

        # 运行测试
        self.run_latency_test()

        # 获取内存使用
        current, peak = tracemalloc.get_traced_memory()
        tracemalloc.stop()

        return {
            'current_memory_mb': current / 1024 / 1024,
            'peak_memory_mb': peak / 1024 / 1024
        }

    def generate_optimization_report(self) -> dict:
        """生成优化报告"""
        latency_results = self.run_latency_test()
        memory_results = self.run_memory_test()

        return {
            'latency_performance': latency_results,
            'memory_usage': memory_results,
            'engine_stats': self.engine.execution_stats,
            'recommendations': self._generate_recommendations(latency_results)
        }

    def _generate_recommendations(self, results: dict) -> list:
        """生成优化建议"""
        recommendations = []

        if results['p99_latency_ns'] > 100000:  # 100微秒
            recommendations.append("P99延迟过高，考虑进一步优化热路径代码")

        if results['throughput_millions_per_sec'] < 1:
            recommendations.append("吞吐量偏低，考虑使用并行处理")

        if results['max_latency_ns'] > results['p99_latency_ns'] * 2:
            recommendations.append("存在异常延迟，检查是否有GC或系统调用")

        return recommendations

# 运行测试
if __name__ == "__main__":
    test = HFTPerformanceTest()
    report = test.generate_optimization_report()

    print("高频交易系统性能优化报告")
    print("=" * 50)
    print(f"平均延迟: {report['latency_performance']['avg_latency_ns']/1000:.2f} μs")
    print(f"P99延迟: {report['latency_performance']['p99_latency_ns']/1000:.2f} μs")
    print(f"吞吐量: {report['latency_performance']['throughput_millions_per_sec']:.2f} M/s")
    print(f"峰值内存: {report['memory_usage']['peak_memory_mb']:.2f} MB")

    print("\n优化建议:")
    for rec in report['recommendations']:
        print(f"- {rec}")
```

## 小结

本章深入探讨了NautilusTrader系统的性能优化技术，涵盖了：

1. **性能分析基础**
   - 性能指标收集和分析
   - 内存管理和垃圾回收优化
   - 性能分析工具使用

2. **并发优化**
   - 异步处理和任务调度
   - 批处理和协程池
   - 多进程和共享内存

3. **高频交易优化**
   - 微秒级延迟优化
   - CPU亲和性和实时优先级
   - 低延迟网络通信

4. **实时性能监控**
   - 系统资源监控
   - 应用性能指标
   - 自动告警系统

5. **优化最佳实践**
   - 代码优化清单
   - 自动优化工具
   - 性能基准测试

通过本章的学习，读者掌握了构建高性能交易系统的关键技术，能够将NautilusTrader应用到对延迟敏感的高频交易场景中。

## 参考资料

- [High Performance Python](https://www.oreilly.com/library/view/high-performance-python/9781449361747/)
- [Python Performance Tips](https://wiki.python.org/moin/PythonSpeed/PerformanceTips)
- [Numba高性能计算](https://numba.pydata.org/)
- [Cython性能优化](https://cython.readthedocs.io/)
## 本章可运行代码入口（源码版）

- 入口脚本：`docs/tutorials_cn/code/chapter12/run_ml_examples.py`
- 关键示例：`docs/tutorials_cn/code/chapter12/ml_strategy.py`、`docs/tutorials_cn/code/chapter12/ml_models.py`

依赖策略说明：`sklearn/torch` 为可选能力；缺失时自动切换到 `numpy` 回退模型。

已验证命令：

```bash
uv run --python 3.13 python docs/tutorials_cn/code/chapter12/run_ml_examples.py
```
