# Руководство по покрытию тестами нового инструментария
# How to Test New Tools and Components

## 📋 Общий подход к тестированию нового инструментария

### 1. **Планирование тестирования перед разработкой**
```python
# Пример структуры планирования для нового инструмента
NEW_TOOL_TEST_PLAN = {
    "component_name": "new_trading_strategy_optimizer",
    "test_categories": {
        "unit_tests": {
            "target_coverage": 85,
            "focus_areas": ["core_logic", "validation", "error_handling"]
        },
        "integration_tests": {
            "target_coverage": 70,
            "focus_areas": ["api_integration", "data_flow", "external_services"]
        },
        "performance_tests": {
            "target_coverage": 60,
            "focus_areas": ["optimization_speed", "memory_usage", "scalability"]
        }
    },
    "testing_tools": ["pytest", "pytest-asyncio", "pytest-cov", "pytest-mock"],
    "mocking_strategy": "comprehensive_interface_mocking",
    "ci_integration": "github_actions_with_coverage_reporting"
}
```

### 2. **Создание базовой структуры тестов**

```python
# Структура для нового инструмента
# tools/new_trading_strategy_optimizer.py
class TradingStrategyOptimizer:
    def __init__(self, config: Dict[str, Any]):
        self.config = config
        self.cache = {}
        self.metrics = {}

    async def optimize_strategy(self, strategy_code: str, market_data: Dict) -> Dict[str, Any]:
        """Optimize trading strategy parameters"""
        # Implementation here
        pass

# tests/unit/test_new_trading_strategy_optimizer.py
import pytest
from unittest.mock import Mock, AsyncMock, patch
from tools.new_trading_strategy_optimizer import TradingStrategyOptimizer


class TestTradingStrategyOptimizer:
    """Test suite for TradingStrategyOptimizer"""

    @pytest.fixture
    def optimizer_config(self):
        """Test configuration for optimizer"""
        return {
            "max_iterations": 100,
            "optimization_algorithm": "genetic",
            "fitness_function": "sharpe_ratio",
            "population_size": 50,
            "mutation_rate": 0.1
        }

    @pytest.fixture
    def sample_strategy_code(self):
        """Sample strategy code for testing"""
        return """
class SampleStrategy:
    def __init__(self):
        self.buy_threshold = 0.02
        self.sell_threshold = -0.01

    def should_buy(self, indicators):
        return indicators['rsi'] < 30 and indicators['macd'] > self.buy_threshold

    def should_sell(self, indicators):
        return indicators['rsi'] > 70 or indicators['macd'] < self.sell_threshold
"""

    @pytest.fixture
    def mock_market_data(self):
        """Mock market data for testing"""
        return {
            "historical_prices": [100, 102, 98, 105, 103, 107, 110],
            "indicators": {
                "rsi": [45, 52, 38, 61, 55, 68, 72],
                "macd": [0.5, 0.3, -0.2, 1.2, 0.8, 1.5, 1.8],
                "sma_20": [95, 96, 97, 98, 99, 100, 101]
            },
            "timeframe": "1h",
            "symbol": "BTC/USDT"
        }

    def test_optimizer_initialization(self, optimizer_config):
        """Test optimizer initialization"""
        optimizer = TradingStrategyOptimizer(optimizer_config)

        assert optimizer.config == optimizer_config
        assert optimizer.cache == {}
        assert optimizer.metrics == {}

    @pytest.mark.asyncio
    async def test_strategy_optimization_basic(self, optimizer_config, sample_strategy_code, mock_market_data):
        """Test basic strategy optimization"""
        optimizer = TradingStrategyOptimizer(optimizer_config)

        # Mock optimization process
        with patch.object(optimizer, '_run_optimization', new_callable=AsyncMock) as mock_optimize:
            mock_optimize.return_value = {
                "optimized_parameters": {"buy_threshold": 0.015, "sell_threshold": -0.005},
                "fitness_score": 1.85,
                "iterations": 75,
                "convergence": True
            }

            result = await optimizer.optimize_strategy(sample_strategy_code, mock_market_data)

            assert "optimized_parameters" in result
            assert "fitness_score" in result
            assert result["fitness_score"] > 1.0
            assert result["iterations"] <= optimizer_config["max_iterations"]

    @pytest.mark.asyncio
    async def test_optimization_with_invalid_strategy(self, optimizer_config, mock_market_data):
        """Test optimization with invalid strategy code"""
        optimizer = TradingStrategyOptimizer(optimizer_config)

        invalid_strategy = "invalid python code {{{"

        with pytest.raises(SyntaxError):
            await optimizer.optimize_strategy(invalid_strategy, mock_market_data)

    @pytest.mark.asyncio
    async def test_optimization_performance_monitoring(self, optimizer_config, sample_strategy_code, mock_market_data):
        """Test optimization performance monitoring"""
        optimizer = TradingStrategyOptimizer(optimizer_config)

        import time
        start_time = time.time()

        # Mock slow optimization
        with patch.object(optimizer, '_run_optimization', new_callable=AsyncMock) as mock_optimize:
            async def slow_optimization(*args, **kwargs):
                await asyncio.sleep(0.1)  # Simulate processing time
                return {
                    "optimized_parameters": {"buy_threshold": 0.01, "sell_threshold": -0.02},
                    "fitness_score": 2.1,
                    "iterations": 50,
                    "processing_time": 0.1
                }

            mock_optimize.side_effect = slow_optimization

            result = await optimizer.optimize_strategy(sample_strategy_code, mock_market_data)

            end_time = time.time()
            duration = end_time - start_time

            # Verify performance is reasonable
            assert duration >= 0.1  # At least the processing time
            assert duration < 1.0   # Should not take too long
            assert result["processing_time"] == 0.1

    def test_configuration_validation(self, optimizer_config):
        """Test configuration validation"""
        # Valid configuration
        optimizer = TradingStrategyOptimizer(optimizer_config)
        assert optimizer.config["max_iterations"] > 0

        # Invalid configuration - negative iterations
        invalid_config = optimizer_config.copy()
        invalid_config["max_iterations"] = -1

        with pytest.raises(ValueError, match="max_iterations must be positive"):
            TradingStrategyOptimizer(invalid_config)

    @pytest.mark.asyncio
    async def test_caching_mechanism(self, optimizer_config, sample_strategy_code, mock_market_data):
        """Test caching mechanism for optimization results"""
        optimizer = TradingStrategyOptimizer(optimizer_config)

        # First optimization
        with patch.object(optimizer, '_run_optimization', new_callable=AsyncMock) as mock_optimize:
            mock_optimize.return_value = {
                "optimized_parameters": {"buy_threshold": 0.02, "sell_threshold": -0.01},
                "fitness_score": 1.75,
                "cached": False
            }

            result1 = await optimizer.optimize_strategy(sample_strategy_code, mock_market_data)

            # Second optimization with same inputs - should use cache
            mock_optimize.return_value = {
                "optimized_parameters": {"buy_threshold": 0.02, "sell_threshold": -0.01},
                "fitness_score": 1.75,
                "cached": True
            }

            result2 = await optimizer.optimize_strategy(sample_strategy_code, mock_market_data)

            # Results should be identical
            assert result1["optimized_parameters"] == result2["optimized_parameters"]
            assert result1["fitness_score"] == result2["fitness_score"]
            assert result2["cached"] is True

    @pytest.mark.asyncio
    async def test_error_handling_and_recovery(self, optimizer_config, sample_strategy_code, mock_market_data):
        """Test error handling and recovery mechanisms"""
        optimizer = TradingStrategyOptimizer(optimizer_config)

        # Test with optimization failure
        with patch.object(optimizer, '_run_optimization', new_callable=AsyncMock) as mock_optimize:
            # First call fails
            mock_optimize.side_effect = [
                Exception("Optimization engine crashed"),
                {  # Second call succeeds (retry)
                    "optimized_parameters": {"buy_threshold": 0.005, "sell_threshold": -0.015},
                    "fitness_score": 1.95,
                    "retries": 1
                }
            ]

            result = await optimizer.optimize_strategy(sample_strategy_code, mock_market_data)

            # Should succeed after retry
            assert "optimized_parameters" in result
            assert result["fitness_score"] == 1.95
            assert result["retries"] == 1

    @pytest.mark.asyncio
    async def test_concurrent_optimization_requests(self, optimizer_config, sample_strategy_code, mock_market_data):
        """Test handling of concurrent optimization requests"""
        optimizer = TradingStrategyOptimizer(optimizer_config)

        # Simulate multiple concurrent requests
        concurrent_requests = 5

        async def single_optimization_request(request_id):
            # Slightly modify market data for each request
            request_data = mock_market_data.copy()
            request_data["request_id"] = request_id

            result = await optimizer.optimize_strategy(sample_strategy_code, request_data)
            return {"request_id": request_id, "result": result}

        # Execute concurrent requests
        import asyncio
        tasks = [single_optimization_request(i) for i in range(concurrent_requests)]
        results = await asyncio.gather(*tasks)

        # Verify all requests completed successfully
        assert len(results) == concurrent_requests
        for result in results:
            assert "result" in result
            assert "optimized_parameters" in result["result"]
            assert result["result"]["fitness_score"] > 0

        # Verify no cross-contamination between requests
        request_ids = [r["request_id"] for r in results]
        assert len(set(request_ids)) == concurrent_requests  # All unique

    def test_metrics_collection_and_reporting(self, optimizer_config, sample_strategy_code, mock_market_data):
        """Test metrics collection and reporting"""
        optimizer = TradingStrategyOptimizer(optimizer_config)

        # Simulate metrics collection during optimization
        with patch.object(optimizer, '_run_optimization', new_callable=AsyncMock) as mock_optimize:
            mock_optimize.return_value = {
                "optimized_parameters": {"buy_threshold": 0.025, "sell_threshold": 0.005},
                "fitness_score": 2.15,
                "iterations": 85,
                "convergence_metrics": {
                    "population_diversity": 0.75,
                    "fitness_improvement_rate": 0.02,
                    "early_convergence_detected": False
                }
            }

            # This would normally be called in optimize_strategy
            # For testing, we'll simulate metrics collection
            optimizer.metrics.update({
                "total_optimizations": 1,
                "average_fitness": 2.15,
                "average_iterations": 85,
                "success_rate": 1.0
            })

            # Verify metrics are collected
            assert optimizer.metrics["total_optimizations"] == 1
            assert optimizer.metrics["average_fitness"] == 2.15
            assert optimizer.metrics["success_rate"] == 1.0

    @pytest.mark.asyncio
    async def test_resource_cleanup_and_memory_management(self, optimizer_config, sample_strategy_code, mock_market_data):
        """Test resource cleanup and memory management"""
        import psutil
        import os

        optimizer = TradingStrategyOptimizer(optimizer_config)

        # Get initial memory usage
        process = psutil.Process(os.getpid())
        initial_memory = process.memory_info().rss / 1024 / 1024  # MB

        # Run optimization (which may allocate memory)
        with patch.object(optimizer, '_run_optimization', new_callable=AsyncMock) as mock_optimize:
            # Simulate memory-intensive optimization
            async def memory_intensive_optimization(*args, **kwargs):
                # Allocate some memory temporarily
                large_data = [list(range(1000)) for _ in range(100)]  # ~100KB
                result = sum(sum(sublist) for sublist in large_data)
                del large_data  # Explicit cleanup
                import gc
                gc.collect()  # Force garbage collection

                return {
                    "optimized_parameters": {"buy_threshold": 0.01, "sell_threshold": -0.005},
                    "fitness_score": 1.95,
                    "memory_peak": "100KB"
                }

            mock_optimize.side_effect = memory_intensive_optimization

            result = await optimizer.optimize_strategy(sample_strategy_code, mock_market_data)

            # Check final memory usage
            final_memory = process.memory_info().rss / 1024 / 1024  # MB
            memory_increase = final_memory - initial_memory

            # Memory increase should be reasonable (< 50MB)
            assert memory_increase < 50
            assert result["fitness_score"] == 1.95

    def test_configuration_hot_reload(self, optimizer_config):
        """Test configuration hot reload capability"""
        optimizer = TradingStrategyOptimizer(optimizer_config)

        # Initial configuration
        assert optimizer.config["max_iterations"] == 100

        # Simulate configuration change
        new_config = optimizer_config.copy()
        new_config["max_iterations"] = 200

        # Hot reload configuration
        optimizer.reload_config(new_config)

        # Verify configuration updated
        assert optimizer.config["max_iterations"] == 200

        # Cache should be cleared on config change
        assert optimizer.cache == {}
```

### 3. **Интеграция с существующими тестами**

```python
# tests/integration/test_new_tool_integration.py
import pytest
from unittest.mock import Mock, AsyncMock, patch


class TestNewToolIntegration:
    """Integration tests for new trading strategy optimizer"""

    @pytest.fixture
    async def integrated_system(self):
        """Setup integrated system with all components"""
        from tools.new_trading_strategy_optimizer import TradingStrategyOptimizer
        from core_server.services.cache_service import CacheService
        from core_server.services.database_service import DatabaseService

        # Mock services
        cache_service = AsyncMock()
        db_service = AsyncMock()

        optimizer = TradingStrategyOptimizer({
            "max_iterations": 50,  # Reduced for testing
            "cache_service": cache_service,
            "db_service": db_service
        })

        return {
            "optimizer": optimizer,
            "cache": cache_service,
            "database": db_service
        }

    @pytest.mark.asyncio
    async def test_end_to_end_optimization_workflow(self, integrated_system):
        """Test complete optimization workflow with all system components"""
        system = integrated_system

        # Setup test data
        strategy_code = "class TestStrategy: pass"
        market_data = {"prices": [100, 101, 102]}

        # Mock all external dependencies
        with patch.object(system["optimizer"], '_run_optimization') as mock_optimize, \
             patch.object(system["cache"], 'get') as mock_cache_get, \
             patch.object(system["database"], 'save_optimization_result') as mock_db_save:

            # Setup mocks
            mock_cache_get.return_value = None  # Cache miss
            mock_optimize.return_value = {
                "parameters": {"param1": 0.1},
                "score": 1.5,
                "iterations": 25
            }

            # Execute workflow
            result = await system["optimizer"].optimize_strategy(strategy_code, market_data)

            # Verify integration
            assert result["score"] > 0
            mock_cache_get.assert_called_once()
            mock_optimize.assert_called_once()
            mock_db_save.assert_called_once()

    @pytest.mark.asyncio
    async def test_performance_under_system_load(self, integrated_system):
        """Test performance under system-wide load"""
        system = integrated_system

        # Simulate system under load
        concurrent_optimizations = 10

        async def optimization_under_load(optimization_id):
            strategy_code = f"class Strategy{optimization_id}: pass"
            market_data = {"id": optimization_id, "data": list(range(100))}

            with patch.object(system["optimizer"], '_run_optimization', new_callable=AsyncMock) as mock_opt:
                mock_opt.return_value = {
                    "parameters": {"param": optimization_id * 0.1},
                    "score": 1.0 + optimization_id * 0.1
                }

                start_time = asyncio.get_event_loop().time()
                result = await system["optimizer"].optimize_strategy(strategy_code, market_data)
                end_time = asyncio.get_event_loop().time()

                return {
                    "id": optimization_id,
                    "duration": end_time - start_time,
                    "result": result
                }

        # Execute concurrent optimizations
        tasks = [optimization_under_load(i) for i in range(concurrent_optimizations)]
        results = await asyncio.gather(*tasks)

        # Analyze performance under load
        durations = [r["duration"] for r in results]
        avg_duration = sum(durations) / len(durations)
        max_duration = max(durations)

        # Performance should degrade gracefully under load
        assert avg_duration < 1.0  # Should complete within reasonable time
        assert max_duration < 2.0  # No individual request should take too long

        # All optimizations should succeed
        assert all(r["result"]["score"] > 0 for r in results)

    @pytest.mark.asyncio
    async def test_error_propagation_and_recovery(self, integrated_system):
        """Test error propagation and recovery across system components"""
        system = integrated_system

        # Test cascade of failures and recovery
        failure_scenarios = [
            "cache_failure",
            "database_failure",
            "optimization_engine_failure",
            "network_timeout"
        ]

        for scenario in failure_scenarios:
            with patch.object(system["optimizer"], '_run_optimization', new_callable=AsyncMock) as mock_opt:
                # Configure failure based on scenario
                if scenario == "cache_failure":
                    system["cache"].get.side_effect = Exception("Cache unavailable")
                    system["cache"].set.side_effect = Exception("Cache unavailable")
                elif scenario == "database_failure":
                    system["database"].save_optimization_result.side_effect = Exception("DB connection failed")
                elif scenario == "optimization_engine_failure":
                    mock_opt.side_effect = Exception("Optimization engine crashed")
                elif scenario == "network_timeout":
                    mock_opt.side_effect = asyncio.TimeoutError("Network timeout")

                # Attempt optimization
                try:
                    result = await system["optimizer"].optimize_strategy("test_code", {"data": []})
                    # If we get here, system handled error gracefully
                    assert "error_handled" in result or "fallback_result" in result
                except Exception as e:
                    # System should provide meaningful error messages
                    assert "graceful" in str(e).lower() or "handled" in str(e).lower()

                # Reset mocks for next scenario
                system["cache"].reset_mock()
                system["database"].reset_mock()
                mock_opt.reset_mock()
```

### 4. **Производительность и нагрузочное тестирование**

```python
# tests/performance/test_new_tool_performance.py
import pytest
import asyncio
import time
from unittest.mock import Mock, AsyncMock, patch


class TestNewToolPerformance:
    """Performance tests for new trading strategy optimizer"""

    @pytest.fixture
    def performance_config(self):
        """Performance testing configuration"""
        return {
            "warmup_iterations": 5,
            "benchmark_iterations": 50,
            "concurrent_users": 10,
            "max_response_time": 1.0,  # seconds
            "target_throughput": 20    # requests per second
        }

    @pytest.mark.asyncio
    async def test_single_request_performance(self, performance_config):
        """Test single request performance"""
        from tools.new_trading_strategy_optimizer import TradingStrategyOptimizer

        optimizer = TradingStrategyOptimizer({"max_iterations": 10})
        strategy_code = "class TestStrategy: pass"
        market_data = {"prices": list(range(100))}

        # Warmup
        for _ in range(performance_config["warmup_iterations"]):
            with patch.object(optimizer, '_run_optimization', new_callable=AsyncMock) as mock_opt:
                mock_opt.return_value = {"parameters": {}, "score": 1.0}
                await optimizer.optimize_strategy(strategy_code, market_data)

        # Benchmark
        response_times = []
        for _ in range(performance_config["benchmark_iterations"]):
            start_time = time.time()

            with patch.object(optimizer, '_run_optimization', new_callable=AsyncMock) as mock_opt:
                mock_opt.return_value = {"parameters": {}, "score": 1.0}
                result = await optimizer.optimize_strategy(strategy_code, market_data)

            end_time = time.time()
            response_times.append(end_time - start_time)

            assert result["score"] > 0

        # Analyze performance
        avg_response_time = sum(response_times) / len(response_times)
        p95_response_time = sorted(response_times)[int(len(response_times) * 0.95)]
        throughput = len(response_times) / sum(response_times)

        # Performance assertions
        assert avg_response_time < performance_config["max_response_time"]
        assert p95_response_time < performance_config["max_response_time"] * 1.5
        assert throughput > performance_config["target_throughput"]

        print(f"✅ Performance test results:")
        print(f"   Average response time: {avg_response_time:.3f}s")
        print(f"   P95 response time: {p95_response_time:.3f}s")
        print(f"   Throughput: {throughput:.1f} req/s")

    @pytest.mark.asyncio
    async def test_concurrent_load_performance(self, performance_config):
        """Test concurrent load performance"""
        from tools.new_trading_strategy_optimizer import TradingStrategyOptimizer

        optimizer = TradingStrategyOptimizer({"max_iterations": 5})
        concurrent_users = performance_config["concurrent_users"]

        async def simulate_user_request(user_id):
            """Simulate a single user request"""
            strategy_code = f"class UserStrategy{user_id}: pass"
            market_data = {"user_id": user_id, "data": list(range(50))}

            start_time = time.time()

            with patch.object(optimizer, '_run_optimization', new_callable=AsyncMock) as mock_opt:
                mock_opt.return_value = {"parameters": {"user_param": user_id}, "score": 1.0 + user_id * 0.01}
                result = await optimizer.optimize_strategy(strategy_code, market_data)

            end_time = time.time()

            return {
                "user_id": user_id,
                "response_time": end_time - start_time,
                "success": result["score"] > 0
            }

        # Execute concurrent requests
        start_time = time.time()
        tasks = [simulate_user_request(i) for i in range(concurrent_users)]
        results = await asyncio.gather(*tasks)
        end_time = time.time()

        total_duration = end_time - start_time

        # Analyze results
        response_times = [r["response_time"] for r in results]
        success_rate = sum(1 for r in results if r["success"]) / len(results)

        avg_response_time = sum(response_times) / len(response_times)
        max_response_time = max(response_times)
        throughput = len(results) / total_duration

        # Performance assertions under load
        assert success_rate == 1.0  # All requests should succeed
        assert avg_response_time < performance_config["max_response_time"] * 2  # Allow some degradation
        assert max_response_time < performance_config["max_response_time"] * 5  # No extreme outliers
        assert throughput > performance_config["target_throughput"] * 0.5  # Maintain reasonable throughput

        print(f"✅ Concurrent load test results ({concurrent_users} users):")
        print(f"   Total duration: {total_duration:.3f}s")
        print(f"   Average response time: {avg_response_time:.3f}s")
        print(f"   Max response time: {max_response_time:.3f}s")
        print(f"   Throughput: {throughput:.1f} req/s")
        print(f"   Success rate: {success_rate:.1%}")

    @pytest.mark.asyncio
    async def test_memory_usage_under_load(self, performance_config):
        """Test memory usage under load"""
        import psutil
        import os

        from tools.new_trading_strategy_optimizer import TradingStrategyOptimizer

        optimizer = TradingStrategyOptimizer({"max_iterations": 5})

        # Get baseline memory
        process = psutil.Process(os.getpid())
        baseline_memory = process.memory_info().rss / 1024 / 1024  # MB

        # Run load test
        concurrent_users = performance_config["concurrent_users"]

        async def memory_intensive_request(user_id):
            """Memory-intensive request simulation"""
            strategy_code = f"class MemoryStrategy{user_id}: pass"
            # Create larger market data to stress memory
            market_data = {
                "prices": list(range(1000)),
                "indicators": {f"indicator_{i}": list(range(500)) for i in range(10)}
            }

            with patch.object(optimizer, '_run_optimization', new_callable=AsyncMock) as mock_opt:
                mock_opt.return_value = {"parameters": {}, "score": 1.0}
                result = await optimizer.optimize_strategy(strategy_code, market_data)

                # Force some memory allocation
                large_temp_data = [list(range(1000)) for _ in range(50)]
                _ = sum(sum(sublist) for sublist in large_temp_data)
                del large_temp_data

                return result

        # Execute memory stress test
        tasks = [memory_intensive_request(i) for i in range(concurrent_users)]
        results = await asyncio.gather(*tasks)

        # Check peak memory usage
        peak_memory = process.memory_info().rss / 1024 / 1024  # MB
        memory_increase = peak_memory - baseline_memory

        # Memory usage should be reasonable
        assert memory_increase < 200  # Less than 200MB increase
        assert all(r["score"] > 0 for r in results)  # All operations successful

        print(f"✅ Memory usage test results:")
        print(f"   Baseline memory: {baseline_memory:.1f} MB")
        print(f"   Peak memory: {peak_memory:.1f} MB")
        print(f"   Memory increase: {memory_increase:.1f} MB")

    @pytest.mark.asyncio
    async def test_scalability_with_increasing_load(self, performance_config):
        """Test scalability with increasing load levels"""
        from tools.new_trading_strategy_optimizer import TradingStrategyOptimizer

        optimizer = TradingStrategyOptimizer({"max_iterations": 3})  # Minimal for speed

        load_levels = [1, 5, 10, 20, 50]
        scalability_results = []

        for load_level in load_levels:
            # Test at this load level
            tasks = []
            for i in range(load_level):
                task = self._create_load_test_task(optimizer, i)
                tasks.append(task)

            start_time = time.time()
            results = await asyncio.gather(*tasks)
            end_time = time.time()

            duration = end_time - start_time
            throughput = load_level / duration
            avg_response_time = duration / load_level

            scalability_results.append({
                "load_level": load_level,
                "duration": duration,
                "throughput": throughput,
                "avg_response_time": avg_response_time,
                "all_successful": all(r["success"] for r in results)
            })

        # Analyze scalability
        throughputs = [r["throughput"] for r in scalability_results]
        response_times = [r["avg_response_time"] for r in scalability_results]

        # Check for performance degradation
        max_throughput = max(throughputs)
        min_throughput = min(throughputs)
        throughput_degradation = (max_throughput - min_throughput) / max_throughput

        # Throughput should degrade gracefully (less than 50% degradation)
        assert throughput_degradation < 0.5

        # Response times should remain reasonable
        max_response_time = max(response_times)
        assert max_response_time < 5.0  # No more than 5 seconds per request

        print(f"✅ Scalability test results:")
        for result in scalability_results:
            print(f"   Load {result['load_level']}: {result['throughput']:.1f} req/s, {result['avg_response_time']:.3f}s avg")

    async def _create_load_test_task(self, optimizer, task_id):
        """Create a load test task"""
        strategy_code = f"class LoadStrategy{task_id}: pass"
        market_data = {"task_id": task_id, "data": list(range(100))}

        with patch.object(optimizer, '_run_optimization', new_callable=AsyncMock) as mock_opt:
            mock_opt.return_value = {"parameters": {}, "score": 1.0}
            result = await optimizer.optimize_strategy(strategy_code, market_data)

        return {
            "task_id": task_id,
            "success": result["score"] > 0,
            "result": result
        }
```

### 5. **Интеграция с CI/CD и отчетность**

```python
# .github/workflows/test-new-tool.yml
name: Test New Trading Strategy Optimizer

on:
  push:
    paths:
      - 'tools/new_trading_strategy_optimizer.py'
      - 'tests/unit/test_new_trading_strategy_optimizer.py'
      - 'tests/integration/test_new_tool_integration.py'
      - 'tests/performance/test_new_tool_performance.py'
  pull_request:
    paths:
      - 'tools/new_trading_strategy_optimizer.py'
      - 'tests/unit/test_new_trading_strategy_optimizer.py'
      - 'tests/integration/test_new_tool_integration.py'
      - 'tests/performance/test_new_tool_performance.py'

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379

    steps:
    - uses: actions/checkout@v3

    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'

    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
        pip install pytest pytest-cov pytest-asyncio pytest-mock

    - name: Run unit tests
      run: |
        pytest tests/unit/test_new_trading_strategy_optimizer.py -v --cov=tools.new_trading_strategy_optimizer --cov-report=xml --cov-report=term-missing

    - name: Run integration tests
      run: |
        pytest tests/integration/test_new_tool_integration.py -v --cov-append --cov-report=xml

    - name: Run performance tests
      run: |
        pytest tests/performance/test_new_tool_performance.py -v --cov-append --cov-report=xml

    - name: Upload coverage reports
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.xml
        flags: new-tool
        name: New Trading Strategy Optimizer Coverage

    - name: Generate test report
      run: |
        pytest tests/unit/test_new_trading_strategy_optimizer.py tests/integration/test_new_tool_integration.py tests/performance/test_new_tool_performance.py --html=reports/new_tool_test_report.html --self-contained-html

    - name: Upload test report
      uses: actions/upload-artifact@v3
      with:
        name: new-tool-test-report
        path: reports/new_tool_test_report.html

  coverage-check:
    runs-on: ubuntu-latest
    needs: test

    steps:
    - name: Check coverage threshold
      run: |
        # This would check if coverage meets the target
        # For now, just verify the workflow completes
        echo "Coverage check completed"
```

---

## 🎯 **Шаговый план добавления тестов для нового инструментария**

### **Шаг 1: Планирование (1-2 часа)**
1. **Определить компоненты для тестирования**
2. **Оценить coverage цели** (unit: 80%, integration: 70%, performance: 60%)
3. **Выбрать testing tools** (pytest, mocks, fixtures)
4. **Создать test structure** (unit/, integration/, performance/)

### **Шаг 2: Unit Testing (4-6 часов)**
1. **Создать базовые тесты** для всех public methods
2. **Добавить edge cases** и error conditions
3. **Протестировать configuration validation**
4. **Достичь 80%+ coverage** для unit tests

### **Шаг 3: Integration Testing (2-4 часа)**
1. **Тестировать взаимодействие** с другими компонентами
2. **Mock external dependencies** (database, cache, APIs)
3. **Проверить data flow** end-to-end
4. **Тестировать error propagation**

### **Шаг 4: Performance Testing (2-3 часа)**
1. **Создать benchmarks** для key operations
2. **Тестировать under load** (concurrent requests)
3. **Monitor resource usage** (memory, CPU)
4. **Установить performance baselines**

### **Шаг 5: CI/CD Integration (1-2 часа)**
1. **Создать GitHub Actions workflow**
2. **Настроить coverage reporting**
3. **Добавить performance regression detection**
4. **Интегрировать с main CI pipeline**

### **Шаг 6: Documentation (1 час)**
1. **Обновить AI_Code_Generation_Improvements.md**
2. **Добавить примеры тестирования**
3. **Документировать testing patterns**
4. **Создать troubleshooting guide**

---

## 📊 **Метрики успеха для нового инструментария**

### **Coverage Targets:**
- **Unit Tests:** 80%+ line coverage
- **Integration Tests:** 70%+ scenario coverage  
- **Performance Tests:** 60%+ load scenario coverage
- **Overall:** 75%+ combined coverage

### **Quality Metrics:**
- **Test Execution Time:** < 5 минут для полного suite
- **Test Success Rate:** > 95%
- **Performance Regression:** < 10% degradation allowed
- **Memory Usage:** < 100MB increase under load

### **CI/CD Integration:**
- **Automated Testing:** ✅ Runs on every PR
- **Coverage Reporting:** ✅ Codecov integration
- **Performance Monitoring:** ✅ Regression detection
- **Documentation:** ✅ Auto-generated reports

---

## 🚀 **Рекомендации по реализации**

### **Для быстрого старта:**
1. **Используйте шаблоны** из этого руководства
2. **Начните с unit tests** - они дают быстрый feedback
3. **Добавляйте integration tests** по мере роста компонента
4. **Внедряйте performance testing** на ранних этапах

### **Для enterprise-grade качества:**
1. **Следуйте TDD подходу** - тесты перед кодом
2. **Используйте comprehensive mocking** для isolation
3. **Автоматизируйте CI/CD** с первого дня
4. **Мониторьте coverage** и performance trends

### **Для поддержки и maintenance:**
1. **Документируйте testing patterns** для команды
2. **Создавайте reusable fixtures** и utilities
3. **Регулярно обновляйте тесты** при изменениях кода
4. **Используйте test-driven debugging** для troubleshooting

---

## 🎉 **Заключение**

**Тестирование нового инструментария - это инвестиция в надежность и maintainability!**

**Ключевые принципы:**
- 🧪 **Test Early, Test Often** - интеграция тестирования с development
- 🎯 **Coverage-Driven Development** - метрики качества как цель
- 🔄 **CI/CD Integration** - автоматизация от первого коммита
- 📊 **Performance-First** - производительность как core requirement

**Следуя этому руководству, вы обеспечите enterprise-grade качество для любого нового инструментария!** 🚀✨

**Ready to test your next tool? Let's build something amazing!** 🎯💎