# Chapter 12: Performance Optimization

## Learning Objectives

- Understand NautilusTrader's performance bottlenecks and optimization strategies
- Master memory management and garbage collection optimization techniques
- Learn to use performance profiling tools for bottleneck identification
- Implement low-latency, high-throughput trading systems

## 12.1 Performance Analysis Fundamentals

### 12.1.1 Performance Metrics

```python
import cProfile
import pstats
import io
from functools import wraps
from time import perf_counter_ns

# Performance metrics collector
class PerformanceMetrics:
    """Performance metrics collector"""

    def __init__(self):
        self.metrics = {
            'latency': [],       # Latency distribution
            'throughput': [],    # Throughput
            'memory_usage': [],  # Memory usage
            'cpu_usage': [],     # CPU usage
            'event_counts': {}   # Event counts
        }
        self.start_time = None

    def start_recording(self):
        """Start recording"""
        self.start_time = perf_counter_ns()

    def record_latency(self, event_type: str, latency_ns: int):
        """Record latency"""
        if event_type not in self.metrics['latency']:
            self.metrics['latency'][event_type] = []
        self.metrics['latency'][event_type].append(latency_ns)

    def record_event(self, event_type: str):
        """Record event"""
        if event_type not in self.metrics['event_counts']:
            self.metrics['event_counts'][event_type] = 0
        self.metrics['event_counts'][event_type] += 1

    def get_percentile(self, latencies: list, percentile: float) -> float:
        """Calculate percentile"""
        if not latencies:
            return 0.0
        latencies.sort()
        index = int(len(latencies) * percentile / 100)
        return latencies[min(index, len(latencies) - 1)]

    def generate_report(self) -> dict:
        """Generate performance report"""
        report = {
            'total_runtime_ns': perf_counter_ns() - self.start_time,
            'event_summary': self.metrics['event_counts']
        }

        # Latency statistics
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

# Performance decorator
def profile_performance(metrics: PerformanceMetrics, event_type: str):
    """Performance profiling decorator"""
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

# Usage example
from nautilus_trader.adapters.betfair.providers import BetfairInstrumentProvider
from nautilus_trader.model.identifiers import InstrumentId
from nautilus_trader.model.objects import Price

class OptimizedInstrumentProvider(BetfairInstrumentProvider):
    """Optimized market data provider"""

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.metrics = PerformanceMetrics()
        self.metrics.start_recording()
        self._price_cache = {}  # Price cache
        self._update_queue = []  # Update queue

    @profile_performance(metrics=None, event_type="quote_tick")
    def process_quote_tick(self, tick):
        """Process quote tick (optimized version)"""
        # Use cache to reduce redundant calculations
        instrument_id = tick.instrument_id
        if instrument_id not in self._price_cache:
            self._price_cache[instrument_id] = {
                'bid': None,
                'ask': None,
                'last_update': 0
            }

        # Batch process updates
        self._update_queue.append(tick)
        if len(self._update_queue) >= 100:  # Batch size
            self._process_batch()

    def _process_batch(self):
        """Batch process updates"""
        start = perf_counter_ns()

        for tick in self._update_queue:
            self._update_price_cache(tick)

        self._update_queue.clear()

        # Record batch processing performance
        processing_time = perf_counter_ns() - start
        self.metrics.record_latency('batch_process', processing_time)
```

### 12.1.2 Memory Optimization

```python
import gc
import psutil
import numpy as np
from typing import List, Dict
from dataclasses import dataclass

# Memory monitor
class MemoryMonitor:
    """Memory usage monitor"""

    def __init__(self):
        self.process = psutil.Process()
        self.baseline_memory = self.get_memory_usage()
        self.peak_memory = self.baseline_memory
        self.memory_history = []

    def get_memory_usage(self) -> float:
        """Get current memory usage (MB)"""
        return self.process.memory_info().rss / 1024 / 1024

    def record_memory(self):
        """Record memory usage"""
        current = self.get_memory_usage()
        self.peak_memory = max(self.peak_memory, current)
        self.memory_history.append(current)

    def get_memory_stats(self) -> dict:
        """Get memory statistics"""
        return {
            'baseline_mb': self.baseline_memory,
            'peak_mb': self.peak_memory,
            'current_mb': self.get_memory_usage(),
            'increase_mb': self.get_memory_usage() - self.baseline_memory,
            'avg_mb': np.mean(self.memory_history) if self.memory_history else 0
        }

# Object pool pattern
class ObjectPool:
    """Object pool to reduce object creation overhead"""

    def __init__(self, create_func, reset_func=None, max_size=1000):
        self.create_func = create_func
        self.reset_func = reset_func
        self.max_size = max_size
        self._pool = []
        self._created_count = 0
        self._reused_count = 0

    def acquire(self):
        """Acquire object"""
        if self._pool:
            self._reused_count += 1
            return self._pool.pop()
        else:
            self._created_count += 1
            return self.create_func()

    def release(self, obj):
        """Release object"""
        if len(self._pool) < self.max_size:
            if self.reset_func:
                self.reset_func(obj)
            self._pool.append(obj)

    def get_stats(self):
        """Get pool statistics"""
        return {
            'pool_size': len(self._pool),
            'created': self._created_count,
            'reused': self._reused_count,
            'reuse_rate': self._reused_count / (self._created_count + self._reused_count)
        }

# Price object pool
def create_price():
    """Create price object"""
    return Price(0, 0)

def reset_price(price):
    """Reset price object"""
    price.raw = 0
    price.precision = 0

price_pool = ObjectPool(create_price, reset_price, max_size=10000)

# Price processor using object pool
class OptimizedPriceProcessor:
    """Optimized price processor"""

    def __init__(self):
        self.price_pool = price_pool
        self.memory_monitor = MemoryMonitor()

    def process_prices(self, raw_prices: List[float]) -> List[Price]:
        """Process price list (using object pool)"""
        self.memory_monitor.record_memory()

        prices = []
        for raw_price in raw_prices:
            price = self.price_pool.acquire()
            price.raw = int(raw_price * 100000)  # Convert to integer
            price.precision = 5
            prices.append(price)

        # Release objects after processing
        # Note: Ensure objects are no longer needed in actual use
        return prices

    def release_prices(self, prices: List[Price]):
        """Release price objects"""
        for price in prices:
            self.price_pool.release(price)

# Zero-copy data processing
class ZeroCopyDataHandler:
    """Zero-copy data processor"""

    def __init__(self):
        self.data_buffer = np.array([], dtype=np.float64)
        self.view_windows = []

    def add_data(self, new_data: np.ndarray):
        """Add data (using memory view)"""
        # Use view to avoid data copying
        if self.data_buffer.size == 0:
            self.data_buffer = new_data
        else:
            # Extend buffer
            new_buffer = np.empty(self.data_buffer.size + new_data.size, dtype=np.float64)
            new_buffer[:self.data_buffer.size] = self.data_buffer
            new_buffer[self.data_buffer.size:] = new_data
            self.data_buffer = new_buffer

    def get_window(self, start: int, end: int) -> np.ndarray:
        """Get data window (return view)"""
        return self.data_buffer[start:end]

    def process_sliding_window(self, window_size: int):
        """Process sliding window"""
        for i in range(len(self.data_buffer) - window_size + 1):
            window = self.data_buffer[i:i+window_size]
            # Process window data (zero-copy)
            yield window

# Memory mapped data processing
import mmap
import os

class MemoryMappedDataLoader:
    """Memory mapped data loader"""

    def __init__(self, file_path: str):
        self.file_path = file_path
        self.file_size = os.path.getsize(file_path)
        self.mmap = None
        self.view = None

    def __enter__(self):
        """Enter context"""
        self.file = open(self.file_path, 'rb')
        self.mmap = mmap.mmap(self.file.fileno(), 0, access=mmap.ACCESS_READ)
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        """Exit context"""
        if self.mmap:
            self.mmap.close()
        if self.file:
            self.file.close()

    def get_view(self, offset: int = 0, size: int = None) -> memoryview:
        """Get memory view"""
        if size is None:
            size = self.file_size - offset
        return memoryview(self.mmap)[offset:offset+size]

    def read_struct(self, offset: int, struct_format: str):
        """Read structured data"""
        import struct
        size = struct.calcsize(struct_format)
        view = self.get_view(offset, size)
        return struct.unpack(struct_format, view.tobytes())
```

## 12.2 Concurrency Optimization

### 12.2.1 Asynchronous Processing

```python
import asyncio
import uvloop
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor
from typing import Callable, Any, List
import time

# Async task scheduler
class AsyncTaskScheduler:
    """Async task scheduler"""

    def __init__(self, max_workers: int = 4):
        self.executor = ThreadPoolExecutor(max_workers=max_workers)
        self.loop = None
        self.tasks = []
        self.task_queue = asyncio.Queue()
        self.result_queue = asyncio.Queue()

    async def initialize(self):
        """Initialize event loop"""
        # Set high-performance event loop
        if hasattr(asyncio, 'set_event_loop_policy'):
            asyncio.set_event_loop_policy(uvloop.EventLoopPolicy())
        self.loop = asyncio.get_running_loop()
        # Start worker coroutine
        asyncio.create_task(self._worker())

    async def submit_task(self, func: Callable, *args, **kwargs) -> str:
        """Submit task"""
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
        """Worker coroutine"""
        while True:
            task = await self.task_queue.get()
            try:
                # Execute CPU-intensive tasks in thread pool
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
        """Get task result"""
        start_time = time.time()
        while True:
            if timeout and time.time() - start_time > timeout:
                raise TimeoutError(f"Task {task_id} timed out")

            # Check result queue
            try:
                result = self.result_queue.get_nowait()
                if result['id'] == task_id:
                    if result['status'] == 'completed':
                        return result['result']
                    else:
                        raise Exception(result['error'])
                else:
                    # Put result back in queue
                    await self.result_queue.put(result)
            except asyncio.QueueEmpty:
                pass

            await asyncio.sleep(0.01)  # Avoid busy waiting

    async def shutdown(self):
        """Shutdown scheduler"""
        await self.task_queue.join()
        self.executor.shutdown(wait=True)

# Batch processor
class AsyncBatchProcessor:
    """Async batch processor"""

    def __init__(self, batch_size: int = 100, timeout: float = 0.1):
        self.batch_size = batch_size
        self.timeout = timeout
        self.buffer = []
        self.last_flush = time.time()
        self.scheduler = AsyncTaskScheduler()
        self.processing_func = None

    async def initialize(self, process_func: Callable):
        """Initialize processor"""
        self.processing_func = process_func
        await self.scheduler.initialize()

    async def add_item(self, item: Any):
        """Add item to batch"""
        self.buffer.append(item)

        # Check if processing is needed
        current_time = time.time()
        should_process = (
            len(self.buffer) >= self.batch_size or
            (self.buffer and current_time - self.last_flush > self.timeout)
        )

        if should_process:
            await self._process_batch()

    async def _process_batch(self):
        """Process batch"""
        if not self.buffer:
            return

        batch = self.buffer.copy()
        self.buffer.clear()
        self.last_flush = time.time()

        # Submit batch processing task
        task_id = await self.scheduler.submit_task(
            self.processing_func,
            batch
        )

        # Can optionally wait for result or process asynchronously
        # await self.scheduler.get_result(task_id)

    async def flush(self):
        """Force process remaining items"""
        await self._process_batch()

    async def shutdown(self):
        """Shutdown processor"""
        await self.flush()
        await self.scheduler.shutdown()

# Coroutine pool
class CoroutinePool:
    """Coroutine pool manager"""

    def __init__(self, pool_size: int = 10):
        self.pool_size = pool_size
        self.semaphore = asyncio.Semaphore(pool_size)
        self.active_tasks = set()
        self.completed_tasks = 0
        self.failed_tasks = 0

    async def submit(self, coro):
        """Submit coroutine"""
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
        """Wait for all tasks to complete"""
        if self.active_tasks:
            await asyncio.wait_for(
                asyncio.gather(*self.active_tasks, return_exceptions=True),
                timeout=timeout
            )

    def get_stats(self):
        """Get statistics"""
        return {
            'active_tasks': len(self.active_tasks),
            'completed_tasks': self.completed_tasks,
            'failed_tasks': self.failed_tasks,
            'pool_size': self.pool_size,
            'available_slots': self.semaphore._value
        }

# Usage example: High-performance data processing pipeline
class HighPerformanceDataPipeline:
    """High-performance data processing pipeline"""

    def __init__(self):
        self.stage1_pool = CoroutinePool(pool_size=20)  # Data reception
        self.stage2_pool = CoroutinePool(pool_size=10)  # Data processing
        self.stage3_pool = CoroutinePool(pool_size=5)   # Result output
        self.batch_processor = AsyncBatchProcessor(batch_size=1000)
        self.metrics = PerformanceMetrics()

    async def initialize(self):
        """Initialize pipeline"""
        await self.batch_processor.initialize(self._process_data_batch)
        self.metrics.start_recording()

    async def process_data_stream(self, data_stream):
        """Process data stream"""
        tasks = []

        async for data_item in data_stream:
            # Stage 1: Fast reception and validation
            task1 = await self.stage1_pool.submit(
                self._validate_data(data_item)
            )

            # Stage 2: Batch processing
            await self.batch_processor.add_item(data_item)

            # Stage 3: Async output
            if task1:
                task3 = await self.stage3_pool.submit(
                    self._output_result(task1)
                )
                tasks.append(task3)

        # Wait for all tasks to complete
        await self.stage1_pool.wait_for_all()
        await self.stage3_pool.wait_for_all()

        # Get performance report
        return self.metrics.generate_report()

    async def _validate_data(self, data):
        """Validate data"""
        # Simple data validation
        if not data:
            return None
        return data

    async def _output_result(self, result):
        """Output result"""
        # Output to database or file
        pass

    def _process_data_batch(self, batch):
        """Batch process data (CPU-intensive)"""
        # This is CPU-intensive computation
        import numpy as np
        data_array = np.array(batch)
        # Perform computation
        result = np.mean(data_array)
        return result
```

### 12.2.2 Multiprocess Optimization

```python
import multiprocessing as mp
from multiprocessing import shared_memory
import numpy as np
from typing import Any, List, Tuple
import pickle

# Shared memory manager
class SharedMemoryManager:
    """Shared memory manager"""

    def __init__(self):
        self.shared_blocks = {}
        self.shapes = {}
        self.dtypes = {}

    def create_array(self, name: str, shape: Tuple, dtype=np.float64):
        """Create shared array"""
        size = np.prod(shape) * np.dtype(dtype).itemsize

        # Create shared memory block
        shm = shared_memory.SharedMemory(create=True, size=size, name=name)
        self.shared_blocks[name] = shm
        self.shapes[name] = shape
        self.dtypes[name] = dtype

        # Create numpy array view
        array = np.ndarray(shape, dtype=dtype, buffer=shm.buf)
        return array

    def get_array(self, name: str) -> np.ndarray:
        """Get shared array"""
        if name not in self.shared_blocks:
            # Try to connect to existing shared memory
            shm = shared_memory.SharedMemory(name=name)
            self.shared_blocks[name] = shm

        shm = self.shared_blocks[name]
        shape = self.shapes.get(name, ())
        dtype = self.dtypes.get(name, np.float64)

        return np.ndarray(shape, dtype=dtype, buffer=shm.buf)

    def cleanup(self, name: str = None):
        """Cleanup shared memory"""
        if name:
            if name in self.shared_blocks:
                self.shared_blocks[name].close()
                self.shared_blocks[name].unlink()
                del self.shared_blocks[name]
        else:
            # Cleanup all
            for name in list(self.shared_blocks.keys()):
                self.cleanup(name)

# Multiprocess data processor
def worker_process(data_name: str, result_name: str,
                   start_idx: int, end_idx: int,
                   worker_id: int) -> dict:
    """Worker process function"""
    shm_manager = SharedMemoryManager()

    # Get shared data
    data_array = shm_manager.get_array(data_name)
    result_array = shm_manager.get_array(result_name)

    # Process data segment
    worker_data = data_array[start_idx:end_idx]

    # Perform computation (example: calculate moving average)
    window_size = 20
    results = []

    for i in range(len(worker_data) - window_size + 1):
        window = worker_data[i:i+window_size]
        ma = np.mean(window)
        results.append(ma)

    # Write results
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
    """Multiprocess data processor"""

    def __init__(self, num_processes: int = None):
        self.num_processes = num_processes or mp.cpu_count()
        self.shm_manager = SharedMemoryManager()
        self.process_pool = None

    def process_large_dataset(self, data: np.ndarray) -> np.ndarray:
        """Process large dataset"""
        data_size = len(data)
        chunk_size = data_size // self.num_processes

        # Create shared memory
        data_name = "shared_data"
        result_name = "shared_result"

        # Share input data
        shared_data = self.shm_manager.create_array(data_name, data.shape, data.dtype)
        shared_data[:] = data[:]

        # Create result array
        result_shape = data.shape
        shared_result = self.shm_manager.create_array(result_name, result_shape, np.float64)

        # Start process pool
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

            # Collect results
            results = [task.get() for task in tasks]

        # Get result array
        result = self.shm_manager.get_array(result_name).copy()

        # Cleanup shared memory
        self.shm_manager.cleanup()

        return result

    def __del__(self):
        """Destructor"""
        if hasattr(self, 'shm_manager'):
            self.shm_manager.cleanup()

# Inter-process communication optimization
class OptimizedIPC:
    """Optimized inter-process communication"""

    def __init__(self):
        self.read_pipe, self.write_pipe = mp.Pipe(duplex=False)
        self.shared_buffer = None

    def create_shared_buffer(self, size: int):
        """Create shared buffer"""
        self.shared_buffer = mp.Array('b', size)

    def send_large_data(self, data: Any):
        """Send large data"""
        # Serialize data
        serialized = pickle.dumps(data)

        # Check if shared buffer is needed
        if len(serialized) > 1024 * 1024:  # 1MB
            if not self.shared_buffer or len(self.shared_buffer) < len(serialized):
                self.create_shared_buffer(len(serialized))

            # Write to shared buffer
            buffer_view = memoryview(self.shared_buffer).cast('B')
            buffer_view[:len(serialized)] = serialized

            # Send metadata
            self.write_pipe.send({
                'type': 'shared_buffer',
                'size': len(serialized)
            })
        else:
            # Send directly
            self.write_pipe.send({
                'type': 'direct',
                'data': serialized
            })

    def receive_large_data(self) -> Any:
        """Receive large data"""
        metadata = self.read_pipe.recv()

        if metadata['type'] == 'shared_buffer':
            # Read from shared buffer
            buffer_view = memoryview(self.shared_buffer).cast('B')
            serialized = bytes(buffer_view[:metadata['size']])
        else:
            # Direct data
            serialized = metadata['data']

        # Deserialize
        return pickle.loads(serialized)

# Parallel computing framework
class ParallelComputeFramework:
    """Parallel computing framework"""

    def __init__(self, num_workers: int = None):
        self.num_workers = num_workers or mp.cpu_count()
        self.workers = []
        self.task_queue = mp.Queue()
        self.result_queue = mp.Queue()
        self.shm_manager = SharedMemoryManager()

    def initialize(self):
        """Initialize worker processes"""
        for i in range(self.num_workers):
            worker = mp.Process(
                target=self._worker_loop,
                args=(i,)
            )
            worker.start()
            self.workers.append(worker)

    def _worker_loop(self, worker_id: int):
        """Worker process loop"""
        while True:
            try:
                task = self.task_queue.get()
                if task is None:  # Stop signal
                    break

                # Execute task
                func_name = task['func']
                args = task.get('args', [])
                kwargs = task.get('kwargs', {})

                # Dynamically import function
                module_name, func_name = func_name.rsplit('.', 1)
                module = __import__(module_name, fromlist=[func_name])
                func = getattr(module, func_name)

                result = func(*args, **kwargs)

                # Send result
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
        """Submit task"""
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
        """Get result"""
        return self.result_queue.get(timeout=timeout)

    def shutdown(self):
        """Shutdown framework"""
        # Send stop signals
        for _ in range(self.num_workers):
            self.task_queue.put(None)

        # Wait for worker processes to end
        for worker in self.workers:
            worker.join()

        # Cleanup shared memory
        self.shm_manager.cleanup()
```

## 12.3 High-Frequency Trading Optimization

### 12.3.1 Microsecond-level Latency Optimization

```python
import time
import ctypes
import numpy as np
from numba import jit, cuda
import cython

# Using Cython to optimize critical paths
# Compilation command: cythonize -i fast_order.pyx

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

# FPGA-accelerated data structures
class FPGAAlignedBuffer:
    """FPGA-aligned buffer"""

    def __init__(self, size: int, alignment: int = 64):
        self.size = size
        self.alignment = alignment
        self.buffer = self._allocate_aligned_memory(size, alignment)

    def _allocate_aligned_memory(self, size: int, alignment: int):
        """Allocate aligned memory"""
        # Use ctypes to allocate aligned memory
        buffer = (ctypes.c_ubyte * size)()
        address = ctypes.addressof(buffer)

        # Calculate aligned address
        aligned_address = (address + alignment - 1) & ~(alignment - 1)

        # Create aligned memory view
        aligned_size = size - (aligned_address - address)
        aligned_buffer = (ctypes.c_ubyte * aligned_size).from_address(aligned_address)

        return aligned_buffer

    def write_atomic(self, offset: int, value: int, size: int = 8):
        """Atomic write"""
        # Atomic operations should be used here
        for i in range(size):
            self.buffer[offset + i] = (value >> (i * 8)) & 0xFF

    def read_atomic(self, offset: int, size: int = 8) -> int:
        """Atomic read"""
        value = 0
        for i in range(size):
            value |= self.buffer[offset + i] << (i * 8)
        return value

# High-frequency trading data structures
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
    """Fast order matching"""

    executed_size = 0.0
    avg_price = 0.0
    total_cost = 0.0

    if is_buy:  # Buy order, match ask side
        for i in range(len(ask_prices)):
            if executed_size >= order_size:
                break

            if ask_prices[i] <= order_price:
                fill_size = min(ask_sizes[i], order_size - executed_size)
                executed_size += fill_size
                total_cost += fill_size * ask_prices[i]

    else:  # Sell order, match bid side
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

# GPU-accelerated risk checking
@cuda.jit
def gpu_risk_check(
    positions: np.ndarray,
    prices: np.ndarray,
    limits: np.ndarray,
    violations: np.ndarray
):
    """GPU parallel risk checking"""

    idx = cuda.grid(1)
    if idx < positions.shape[0]:
        # Calculate current P&L
        pnl = positions[idx] * prices[idx]

        # Check if limit is exceeded
        if abs(pnl) > limits[idx]:
            violations[idx] = 1
        else:
            violations[idx] = 0

class HFTOptimizer:
    """High-frequency trading optimizer"""

    def __init__(self):
        self.tick_processor = None
        self.order_book = None
        self.risk_engine = None
        self.execution_engine = None

    def optimize_tick_to_trade(self):
        """Optimize tick-to-trade latency"""

        # 1. CPU affinity setting
        import os
        os.sched_setaffinity(0, {0})  # Bind to CPU 0

        # 2. Real-time priority
        import psutil
        p = psutil.Process()
        if hasattr(p, 'ionice'):
            p.ionice(psutil.IOPRIO_HIGHEST)  # Highest IO priority
        if hasattr(p, 'nice'):
            p.nice(-20)  # Highest CPU priority

        # 3. Warm up caches
        self._warmup_caches()

        # 4. Disable swap
        self._disable_swap()

        # 5. Optimize network
        self._optimize_network()

    def _warmup_caches(self):
        """Warm up CPU caches"""
        # Pre-allocate all needed memory
        large_arrays = []
        for _ in range(10):
            arr = np.zeros(1000000, dtype=np.float64)
            arr.fill(1.0)
            large_arrays.append(arr)

        # Trigger cache fill
        for arr in large_arrays:
            _ = np.sum(arr)

    def _disable_swap(self):
        """Disable swap"""
        import subprocess
        try:
            subprocess.run(['sudo', 'swapoff', '-a'], check=True)
        except:
            pass  # May require root privileges

    def _optimize_network(self):
        """Optimize network settings"""
        import subprocess
        try:
            # Set network buffers
            subprocess.run([
                'sudo', 'sysctl', '-w', 'net.core.rmem_max=134217728'
            ], check=True)
            subprocess.run([
                'sudo', 'sysctl', '-w', 'net.core.wmem_max=134217728'
            ], check=True)
        except:
            pass

    def measure_latency(self, iterations: int = 10000) -> dict:
        """Measure latency"""
        latencies = []

        for _ in range(iterations):
            start = time.perf_counter_ns()

            # Simulate tick-to-trade operation
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
        """Simulate tick processing"""
        # Simulate data processing
        data = np.random.randn(1000)
        result = np.mean(data)

        # Simulate order processing
        order_book = np.random.rand(1000, 2)
        executed = fast_order_matching(
            order_book[:, 0], order_book[:, 1],
            order_book[:, 0], order_book[:, 1],
            result, 100.0, True
        )

# Low-latency network communication
import socket
import struct

class LowLatencyNetwork:
    """Low-latency network communication"""

    def __init__(self, host: str, port: int):
        self.host = host
        self.port = port
        self.socket = None
        self.buffer_size = 65536  # 64KB

    def connect(self):
        """Establish connection"""
        # Create TCP socket
        self.socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

        # Set socket options
        self.socket.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)
        self.socket.setsockopt(socket.SOL_SOCKET, socket.SO_SNDBUF, self.buffer_size)
        self.socket.setsockopt(socket.SOL_SOCKET, socket.SO_RCVBUF, self.buffer_size)

        # Connect
        self.socket.connect((self.host, self.port))

    def send_market_data(self, data: dict):
        """Send market data"""
        # Custom binary protocol
        message = self._encode_market_data(data)
        self.socket.sendall(message)

    def _encode_market_data(self, data: dict) -> bytes:
        """Encode market data"""
        # Simple binary encoding
        fmt = '!QdI'  # timestamp(8) + price(8) + size(4)
        return struct.pack(fmt,
                          int(data['timestamp'] * 1e9),
                          data['price'],
                          int(data['size']))

    def close(self):
        """Close connection"""
        if self.socket:
            self.socket.close()
```

## 12.4 Real-time Performance Monitoring

### 12.4.1 System Monitoring

```python
import psutil
import GPUtil
import nvidia_ml_py3 as nvml
from prometheus_client import CollectorRegistry, Gauge, start_http_server
import time

class SystemMonitor:
    """System resource monitor"""

    def __init__(self):
        self.registry = CollectorRegistry()
        self.setup_metrics()
        self.monitoring = False

    def setup_metrics(self):
        """Setup monitoring metrics"""
        # CPU metrics
        self.cpu_usage = Gauge(
            'cpu_usage_percent',
            'CPU usage percentage',
            registry=self.registry
        )
        self.cpu_temp = Gauge(
            'cpu_temperature_celsius',
            'CPU temperature',
            registry=self.registry
        )

        # Memory metrics
        self.memory_usage = Gauge(
            'memory_usage_percent',
            'Memory usage percentage',
            registry=self.registry
        )
        self.memory_available = Gauge(
            'memory_available_bytes',
            'Available memory bytes',
            registry=self.registry
        )

        # Network metrics
        self.network_bytes_sent = Gauge(
            'network_bytes_sent_total',
            'Network bytes sent',
            registry=self.registry
        )
        self.network_bytes_recv = Gauge(
            'network_bytes_recv_total',
            'Network bytes received',
            registry=self.registry
        )

        # GPU metrics
        self.gpu_usage = Gauge(
            'gpu_usage_percent',
            'GPU usage percentage',
            labelnames=['gpu_id'],
            registry=self.registry
        )
        self.gpu_memory_usage = Gauge(
            'gpu_memory_usage_percent',
            'GPU memory usage percentage',
            labelnames=['gpu_id'],
            registry=self.registry
        )

    def start_monitoring(self, port: int = 8000):
        """Start monitoring"""
        # Start Prometheus HTTP server
        start_http_server(port, registry=self.registry)
        self.monitoring = True

        # Initialize counters
        self.initial_net_io = psutil.net_io_counters()

        # Start monitoring loop
        import threading
        self.monitor_thread = threading.Thread(target=self._monitor_loop)
        self.monitor_thread.daemon = True
        self.monitor_thread.start()

    def _monitor_loop(self):
        """Monitoring loop"""
        while self.monitoring:
            # Update CPU metrics
            self.cpu_usage.set(psutil.cpu_percent())

            # CPU temperature (requires lm-sensors)
            try:
                temps = psutil.sensors_temperatures()
                if 'coretemp' in temps:
                    self.cpu_temp.set(temps['coretemp'][0].current)
            except:
                pass

            # Update memory metrics
            memory = psutil.virtual_memory()
            self.memory_usage.set(memory.percent)
            self.memory_available.set(memory.available)

            # Update network metrics
            net_io = psutil.net_io_counters()
            self.network_bytes_sent.set(net_io.bytes_sent)
            self.network_bytes_recv.set(net_io.bytes_recv)

            # Update GPU metrics
            try:
                gpus = GPUtil.getGPUs()
                for gpu in gpus:
                    self.gpu_usage.labels(gpu_id=gpu.id).set(gpu.load * 100)
                    self.gpu_memory_usage.labels(gpu_id=gpu.id).set(
                        gpu.memoryUtil * 100
                    )
            except:
                pass

            time.sleep(1)  # Update every second

    def stop_monitoring(self):
        """Stop monitoring"""
        self.monitoring = False

# Application performance monitoring
class ApplicationMonitor:
    """Application performance monitor"""

    def __init__(self):
        self.metrics = {}
        self.timers = {}
        self.counters = {}
        self.histograms = {}

    def timer(self, name: str):
        """Timer decorator"""
        def decorator(func):
            def wrapper(*args, **kwargs):
                start_time = time.perf_counter()
                result = func(*args, **kwargs)
                duration = time.perf_counter() - start_time

                # Record timing
                if name not in self.metrics:
                    self.metrics[name] = []
                self.metrics[name].append(duration)

                return result
            return wrapper
        return decorator

    def counter(self, name: str):
        """Counter decorator"""
        def decorator(func):
            def wrapper(*args, **kwargs):
                result = func(*args, **kwargs)

                # Increment count
                if name not in self.counters:
                    self.counters[name] = 0
                self.counters[name] += 1

                return result
            return wrapper
        return decorator

    def histogram(self, name: str, value: float):
        """Record histogram data"""
        if name not in self.histograms:
            self.histograms[name] = []
        self.histograms[name].append(value)

    def get_percentiles(self, name: str) -> dict:
        """Get percentiles"""
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
        """Generate monitoring report"""
        report = {
            'counters': self.counters,
            'timers': {}
        }

        for name in self.metrics:
            report['timers'][name] = self.get_percentiles(name)

        return report

# Real-time latency monitoring
class LatencyMonitor:
    """Real-time latency monitor"""

    def __init__(self, window_size: int = 1000):
        self.window_size = window_size
        self.latencies = []
        self.anomalies = []
        self.baseline_latency = None
        self.threshold_multiplier = 3.0

    def record_latency(self, latency_ns: int):
        """Record latency"""
        self.latencies.append(latency_ns)

        # Maintain window size
        if len(self.latencies) > self.window_size:
            self.latencies.pop(0)

        # Detect anomalies
        self._detect_anomaly(latency_ns)

    def _detect_anomaly(self, latency_ns: int):
        """Detect latency anomalies"""
        if len(self.latencies) < 100:  # Need sufficient data
            return

        if not self.baseline_latency:
            # Calculate baseline latency (median)
            self.baseline_latency = np.median(self.latencies[:-10])

        # Check if exceeds threshold
        threshold = self.baseline_latency * self.threshold_multiplier
        if latency_ns > threshold:
            self.anomalies.append({
                'timestamp': time.time(),
                'latency_ns': latency_ns,
                'baseline_ns': self.baseline_latency,
                'threshold_ns': threshold
            })

    def get_real_time_stats(self) -> dict:
        """Get real-time statistics"""
        if not self.latencies:
            return {}

        recent_latencies = self.latencies[-100:]  # Last 100

        return {
            'current_ns': self.latencies[-1],
            'avg_recent_ns': np.mean(recent_latencies),
            'p99_recent_ns': np.percentile(recent_latencies, 99),
            'anomaly_count': len(self.anomalies),
            'anomaly_rate': len(self.anomalies) / len(self.latencies) * 100
        }

# Performance alert system
class PerformanceAlertSystem:
    """Performance alert system"""

    def __init__(self):
        self.alerts = []
        self.rules = []
        self.notification_handlers = []

    def add_rule(self, name: str, metric: str, condition: str, threshold: float):
        """Add alert rule"""
        self.rules.append({
            'name': name,
            'metric': metric,
            'condition': condition,
            'threshold': threshold,
            'last_triggered': 0
        })

    def check_metrics(self, metrics: dict):
        """Check metrics"""
        current_time = time.time()

        for rule in self.rules:
            metric_value = metrics.get(rule['metric'], 0)

            # Evaluate condition
            triggered = False
            if rule['condition'] == '>':
                triggered = metric_value > rule['threshold']
            elif rule['condition'] == '<':
                triggered = metric_value < rule['threshold']
            elif rule['condition'] == '==':
                triggered = metric_value == rule['threshold']

            # Trigger alert
            if triggered and (current_time - rule['last_triggered'] > 60):  # 1 minute cooldown
                alert = {
                    'rule': rule['name'],
                    'metric': rule['metric'],
                    'value': metric_value,
                    'threshold': rule['threshold'],
                    'timestamp': current_time
                }

                self.alerts.append(alert)
                rule['last_triggered'] = current_time

                # Send notification
                self._send_notification(alert)

    def _send_notification(self, alert: dict):
        """Send notification"""
        for handler in self.notification_handlers:
            handler(alert)

    def add_email_handler(self, smtp_server: str, from_email: str, to_emails: list):
        """Add email notification handler"""
        import smtplib
        from email.mime.text import MIMEText

        def email_handler(alert):
            msg = MIMEText(f"""
            Performance Alert Triggered!

            Rule: {alert['rule']}
            Metric: {alert['metric']}
            Current Value: {alert['value']}
            Threshold: {alert['threshold']}
            Time: {time.ctime(alert['timestamp'])}
            """)

            msg['Subject'] = f"Performance Alert: {alert['rule']}"
            msg['From'] = from_email
            msg['To'] = ', '.join(to_emails)

            server = smtplib.SMTP(smtp_server)
            server.send_message(msg)
            server.quit()

        self.notification_handlers.append(email_handler)

    def add_slack_handler(self, webhook_url: str):
        """Add Slack notification handler"""
        import requests

        def slack_handler(alert):
            payload = {
                "text": f"""
                :warning: Performance Alert Triggered!
                Rule: {alert['rule']}
                Metric: {alert['metric']}
                Current Value: {alert['value']}
                Threshold: {alert['threshold']}
                """
            }

            requests.post(webhook_url, json=payload)

        self.notification_handlers.append(slack_handler)
```

## 12.5 Performance Optimization Best Practices

### 12.5.1 Code Optimization Checklist

```python
# Performance optimization checklist
class PerformanceOptimizationChecklist:
    """Performance optimization checklist"""

    @staticmethod
    def memory_optimizations():
        """Memory optimization checklist items"""
        return {
            "Object Pool": "Use object pools to reduce GC pressure",
            "Zero Copy": "Use memory views to avoid data copying",
            "Data Types": "Use appropriate data types (float32 instead of float64)",
            "Pre-allocation": "Pre-allocate array sizes",
            "Lazy Evaluation": "Compute only when needed",
            "Memory Mapping": "Use memory mapping for large files",
            "Circular References": "Avoid circular references"
        }

    @staticmethod
    def cpu_optimizations():
        """CPU optimization checklist items"""
        return {
            "Vectorization": "Use numpy vectorized operations",
            "JIT Compilation": "Use numba JIT compilation for hot paths",
            "Cython": "Rewrite critical code in Cython",
            "Algorithm Complexity": "Optimize algorithm complexity",
            "Cache Friendly": "Optimize data access patterns",
            "Branch Prediction": "Reduce conditional branches",
            "SIMD": "Use SIMD instructions"
        }

    @staticmethod
    def io_optimizations():
        """IO optimization checklist items"""
        return {
            "Batch Processing": "Batch process to reduce IO operations",
            "Async IO": "Use async IO model",
            "Compression": "Compress to reduce transmission",
            "Connection Pool": "Reuse connections",
            "Buffering": "Appropriate buffer sizes",
            "Serialization": "Efficient serialization formats",
            "Prefetching": "Prefetch data"
        }

# Automatic performance optimization tool
class AutoPerformanceOptimizer:
    """Automatic performance optimization tool"""

    def __init__(self, code_path: str):
        self.code_path = code_path
        self.optimizations = []

    def analyze_code(self) -> dict:
        """Analyze code to find optimization points"""
        import ast
        import os

        issues = []

        # Traverse Python files
        for root, dirs, files in os.walk(self.code_path):
            for file in files:
                if file.endswith('.py'):
                    file_path = os.path.join(root, file)
                    with open(file_path, 'r') as f:
                        content = f.read()

                    # Parse AST
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
        """Analyze AST to find performance issues"""
        issues = []

        for node in ast.walk(tree):
            # Check for append in loops
            if isinstance(node, ast.For):
                for inner_node in ast.walk(node):
                    if isinstance(inner_node, ast.Call):
                        if (isinstance(inner_node.func, ast.Attribute) and
                            inner_node.func.attr == 'append'):
                            issues.append({
                                'type': 'list_append_in_loop',
                                'file': file_path,
                                'line': node.lineno,
                                'message': 'list.append in loop can be pre-allocated'
                            })

            # Check for repeated calculations
            if isinstance(node, ast.Call):
                # More complex analysis logic can be added here
                pass

        return issues

    def _generate_recommendations(self, issues: list) -> list:
        """Generate optimization recommendations"""
        recommendations = []

        # Group by issue type
        issue_types = {}
        for issue in issues:
            issue_type = issue['type']
            if issue_type not in issue_types:
                issue_types[issue_type] = []
            issue_types[issue_type].append(issue)

        # Generate recommendations
        for issue_type, type_issues in issue_types.items():
            if issue_type == 'list_append_in_loop':
                recommendations.append({
                    'type': 'list_append_in_loop',
                    'count': len(type_issues),
                    'suggestion': 'Use numpy arrays or pre-allocate list size',
                    'example': '''
# Before optimization
result = []
for x in data:
    result.append(x * 2)

# After optimization
result = [0] * len(data)
for i, x in enumerate(data):
    result[i] = x * 2

# Or use numpy
result = np.array(data) * 2
                    '''
                })

        return recommendations

    def apply_optimizations(self, recommendations: list):
        """Apply optimization recommendations"""
        # Automatic application of optimizations can be implemented here
        pass

# Performance benchmark suite
class PerformanceBenchmarkSuite:
    """Performance benchmark suite"""

    def __init__(self):
        self.benchmarks = {}
        self.results = {}

    def benchmark(self, name: str):
        """Benchmark decorator"""
        def decorator(func):
            def wrapper(*args, **kwargs):
                # Warm up
                for _ in range(10):
                    func(*args, **kwargs)

                # Actual test
                times = []
                for _ in range(100):
                    start = time.perf_counter_ns()
                    result = func(*args, **kwargs)
                    end = time.perf_counter_ns()
                    times.append(end - start)

                # Record results
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
        """Run all benchmarks"""
        print("Benchmark Results:")
        print("-" * 50)

        for name, result in self.results.items():
            print(f"{name}:")
            print(f"  Average: {result['mean_ns']/1000:.2f} μs")
            print(f"  Min: {result['min_ns']/1000:.2f} μs")
            print(f"  Max: {result['max_ns']/1000:.2f} μs")
            print(f"  Std: {result['std_ns']/1000:.2f} μs")
            print()

# Usage example
benchmark = PerformanceBenchmarkSuite()

@benchmark.benchmark("list_creation")
def test_list_creation():
    return [i for i in range(1000)]

@benchmark.benchmark("numpy_array_creation")
def test_numpy_creation():
    return np.arange(1000)

# Run benchmarks
test_list_creation()
test_numpy_creation()
benchmark.run_all()
```

## 12.6 Practical Cases

### 12.6.1 High-Frequency Market Making Strategy Optimization

```python
import numpy as np
from numba import jit, types
from numba.typed import Dict

# High-frequency market making strategy core
@jit(nopython=True, cache=True, parallel=True)
def high_frequency_market_making(
    mid_prices: np.ndarray,
    spreads: np.ndarray,
    inventory: np.ndarray,
    risk_limits: np.ndarray,
    depth: int
) -> tuple:
    """
    High-frequency market making strategy core computation

    Returns:
        (bid_prices, ask_prices, bid_sizes, ask_sizes)
    """
    n = len(mid_prices)
    bid_prices = np.zeros(n)
    ask_prices = np.zeros(n)
    bid_sizes = np.zeros(n)
    ask_sizes = np.zeros(n)

    # Process each time point in parallel
    for i in range(n):
        # Current inventory and risk limits
        current_inventory = inventory[i]
        risk_limit = risk_limits[i]

        # Base spread
        base_spread = spreads[i]

        # Adjust quotes based on inventory
        inventory_skew = current_inventory / risk_limit

        # Calculate bid and ask prices
        bid_price = mid_prices[i] - base_spread * (1 + inventory_skew)
        ask_price = mid_prices[i] + base_spread * (1 - inventory_skew)

        # Adjust quotes based on depth
        if depth > 1:
            # Multi-level quotes
            for level in range(depth):
                level_spread = base_spread * (1 + level * 0.1)
                bid_prices[i * depth + level] = bid_price - level * level_spread
                ask_prices[i * depth + level] = ask_price + level * level_spread

                # Decreasing order sizes
                size = 100 / (level + 1)
                bid_sizes[i * depth + level] = size
                ask_sizes[i * depth + level] = size
        else:
            # Single level quotes
            bid_prices[i] = bid_price
            ask_prices[i] = ask_price
            bid_sizes[i] = 100
            ask_sizes[i] = 100

    return bid_prices, ask_prices, bid_sizes, ask_sizes

# Optimized order book management
class OptimizedOrderBook:
    """Optimized order book manager"""

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
        """Update bid side (JIT optimized)"""
        if level < self.max_levels:
            self.bid_prices[level] = price
            self.bid_sizes[level] = size
            self.bid_count = max(self.bid_count, level + 1)
            self.last_update = time.time_ns()

    @jit(nopython=True)
    def update_ask(self, price: float, size: float, level: int):
        """Update ask side (JIT optimized)"""
        if level < self.max_levels:
            self.ask_prices[level] = price
            self.ask_sizes[level] = size
            self.ask_count = max(self.ask_count, level + 1)
            self.last_update = time.time_ns()

    @jit(nopython=True)
    def get_best_bid(self) -> tuple:
        """Get best bid price"""
        if self.bid_count > 0:
            # Prices sorted from high to low
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
        """Get best ask price"""
        if self.ask_count > 0:
            # Prices sorted from low to high
            best_idx = 0
            best_price = self.ask_prices[0]
            for i in range(1, self.ask_count):
                if self.ask_prices[i] < best_price and self.ask_sizes[i] > 0:
                    best_price = self.ask_prices[i]
                    best_idx = i
            return best_price, self.ask_sizes[best_idx]
        return 0.0, 0.0

# Ultra-low latency execution engine
class UltraLowLatencyEngine:
    """Ultra-low latency execution engine"""

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
        """Process market data (JIT optimized)"""
        # Update order book
        self.order_book.update_bid(bid_price, 100.0, 0)
        self.order_book.update_ask(ask_price, 100.0, 0)

        # Calculate mid price
        mid_price = (bid_price + ask_price) / 2.0
        spread = ask_price - bid_price

        # Get current position
        current_position = self.position_manager.get_position()

        # Risk check
        risk_ok = self.risk_checker.check_position_limit(current_position)

        # Determine quotes
        if risk_ok:
            # Use JIT optimized market making calculation
            bid_prices, ask_prices, bid_sizes, ask_sizes = high_frequency_market_making(
                np.array([mid_price]),
                np.array([spread / 2.0]),
                np.array([current_position]),
                np.array([1000.0]),  # Risk limit
                1  # Depth
            )

            return (
                bid_prices[0], ask_prices[0],
                bid_sizes[0], ask_sizes[0]
            )

        return 0.0, 0.0, 0.0, 0.0

    def execute_order(self, side: str, price: float, size: float) -> bool:
        """Execute order"""
        start_time = time.perf_counter_ns()

        # Fast risk check
        if not self.risk_checker.fast_risk_check(side, size):
            return False

        # Simulate order sending
        success = self._send_order(side, price, size)

        # Update statistics
        end_time = time.perf_counter_ns()
        latency = end_time - start_time

        self.execution_stats['orders_sent'] += 1
        if success:
            self.execution_stats['orders_filled'] += 1
            self.position_manager.update_position(side, size)

        # Update latency statistics
        self._update_latency_stats(latency)

        return success

    def _send_order(self, side: str, price: float, size: float) -> bool:
        """Send order (simulation)"""
        # This should be actual order sending logic
        return True

    def _update_latency_stats(self, latency_ns: int):
        """Update latency statistics"""
        stats = self.execution_stats
        stats['min_latency_ns'] = min(stats['min_latency_ns'], latency_ns)
        stats['max_latency_ns'] = max(stats['max_latency_ns'], latency_ns)

        # Calculate moving average
        n = stats['orders_sent']
        stats['avg_latency_ns'] = (
            (stats['avg_latency_ns'] * (n - 1) + latency_ns) / n
        )

# Fast position manager
@jit(nopython=True)
class FastPositionManager:
    """Fast position manager"""

    def __init__(self):
        self.positions = Dict.empty(key_type=types.unicode_type, value_type=types.float64)
        self.total_position = 0.0

    def update_position(self, symbol: str, quantity: float):
        """Update position"""
        if symbol in self.positions:
            self.positions[symbol] += quantity
        else:
            self.positions[symbol] = quantity
        self.total_position += quantity

    def get_position(self, symbol: str = None) -> float:
        """Get position"""
        if symbol:
            return self.positions.get(symbol, 0.0)
        return self.total_position

# JIT risk checker
@jit(nopython=True)
class JITRiskChecker:
    """JIT optimized risk checker"""

    def __init__(self):
        self.max_position = 1000.0
        self.max_order_size = 100.0
        self.max_notional = 100000.0

    def check_position_limit(self, position: float) -> bool:
        """Check position limit"""
        return abs(position) < self.max_position

    def fast_risk_check(self, side: str, size: float) -> bool:
        """Fast risk check"""
        return size < self.max_order_size

# Performance testing framework
class HFTPerformanceTest:
    """High-frequency trading performance test"""

    def __init__(self):
        self.engine = UltraLowLatencyEngine()
        self.test_data = self._generate_test_data()

    def _generate_test_data(self, n: int = 100000) -> list:
        """Generate test data"""
        np.random.seed(42)
        data = []
        base_price = 100.0

        for i in range(n):
            # Random walk price
            base_price += np.random.randn() * 0.01

            # Bid-ask spread
            spread = np.random.uniform(0.01, 0.05)

            data.append({
                'bid': base_price - spread / 2,
                'ask': base_price + spread / 2,
                'timestamp': i
            })

        return data

    def run_latency_test(self) -> dict:
        """Run latency test"""
        latencies = []

        for tick in self.test_data:
            start = time.perf_counter_ns()

            # Process tick
            bid, ask, bid_size, ask_size = self.engine.process_market_data(
                tick['bid'], tick['ask'], tick['timestamp']
            )

            # Execute order
            if bid > 0:
                self.engine.execute_order('BUY', bid, bid_size)
            if ask > 0:
                self.engine.execute_order('SELL', ask, ask_size)

            end = time.perf_counter_ns()
            latencies.append(end - start)

        # Statistical results
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
        """Run memory test"""
        import tracemalloc

        tracemalloc.start()

        # Run test
        self.run_latency_test()

        # Get memory usage
        current, peak = tracemalloc.get_traced_memory()
        tracemalloc.stop()

        return {
            'current_memory_mb': current / 1024 / 1024,
            'peak_memory_mb': peak / 1024 / 1024
        }

    def generate_optimization_report(self) -> dict:
        """Generate optimization report"""
        latency_results = self.run_latency_test()
        memory_results = self.run_memory_test()

        return {
            'latency_performance': latency_results,
            'memory_usage': memory_results,
            'engine_stats': self.engine.execution_stats,
            'recommendations': self._generate_recommendations(latency_results)
        }

    def _generate_recommendations(self, results: dict) -> list:
        """Generate optimization recommendations"""
        recommendations = []

        if results['p99_latency_ns'] > 100000:  # 100 microseconds
            recommendations.append("P99 latency too high, consider further optimizing hot path code")

        if results['throughput_millions_per_sec'] < 1:
            recommendations.append("Throughput too low, consider using parallel processing")

        if results['max_latency_ns'] > results['p99_latency_ns'] * 2:
            recommendations.append("Abnormal latency exists, check for GC or system calls")

        return recommendations

# Run test
if __name__ == "__main__":
    test = HFTPerformanceTest()
    report = test.generate_optimization_report()

    print("High-Frequency Trading System Performance Optimization Report")
    print("=" * 50)
    print(f"Average Latency: {report['latency_performance']['avg_latency_ns']/1000:.2f} μs")
    print(f"P99 Latency: {report['latency_performance']['p99_latency_ns']/1000:.2f} μs")
    print(f"Throughput: {report['latency_performance']['throughput_millions_per_sec']:.2f} M/s")
    print(f"Peak Memory: {report['memory_usage']['peak_memory_mb']:.2f} MB")

    print("\nOptimization Recommendations:")
    for rec in report['recommendations']:
        print(f"- {rec}")
```

## Summary

This chapter deeply explored the performance optimization techniques of the NautilusTrader system, covering:

1. **Performance Analysis Fundamentals**
   - Performance metrics collection and analysis
   - Memory management and garbage collection optimization
   - Performance profiling tools usage

2. **Concurrency Optimization**
   - Asynchronous processing and task scheduling
   - Batch processing and coroutine pools
   - Multiprocessing and shared memory

3. **High-Frequency Trading Optimization**
   - Microsecond-level latency optimization
   - CPU affinity and real-time priority
   - Low-latency network communication

4. **Real-time Performance Monitoring**
   - System resource monitoring
   - Application performance metrics
   - Automatic alert systems

5. **Optimization Best Practices**
   - Code optimization checklist
   - Automatic optimization tools
   - Performance benchmarking

Through this chapter, readers have mastered the key technologies for building high-performance trading systems and can apply NautilusTrader to latency-sensitive high-frequency trading scenarios.

## References

- [High Performance Python](https://www.oreilly.com/library/view/high-performance-python/9781449361747/)
- [Python Performance Tips](https://wiki.python.org/moin/PythonSpeed/PerformanceTips)
- [Numba High-Performance Computing](https://numba.pydata.org/)
- [Cython Performance Optimization](https://cython.readthedocs.io/)

## Runnable Code Entry (Source Build)

- Entry script: docs/tutorials_cn/code/chapter12/run_ml_examples.py
- Key examples: docs/tutorials_cn/code/chapter12/ml_strategy.py, docs/tutorials_cn/code/chapter12/ml_models.py

Dependency strategy: sklearn/torch are optional; examples fallback to numpy implementations when dependencies are missing.

Verified command:

```bash
uv run --python 3.13 python docs/tutorials_cn/code/chapter12/run_ml_examples.py
```
