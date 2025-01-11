# Python Code Sample

**Author**: Derek McIntire  
**Updated**: January 2024  
**Project Repository**: [derekvmcintire/zwift-processor](https://github.com/derekvmcintire/zwift-processor)

## Overview

This codebase demonstrates using Python for processing Zwift cycling race results, calculating general classifications (GC), and managing race data. It showcases current best practices in Python development, including async operations, type safety, and efficient data processing.

## Key Concepts

- **Async Operations**: Using Python's async/await for efficient API interactions
- **Type Safety**: Leveraging Pydantic models and type hints for robust data validation
- **Modern Data Processing**: Using Pandas for efficient race result calculations
- **Configuration Management**: Environment-based configuration using Pydantic settings
- **Resource Management**: Proper handling of async contexts and file operations
- **Error Handling**: Comprehensive logging and exception management

---

## Code

### **Settings Configuration**

Manages application configuration using environment variables with Pydantic settings validation.

```python
from pydantic_settings import BaseSettings
from pydantic import Field
from pathlib import Path

class Settings(BaseSettings):
    """Application configuration using environment variables.

    Attributes:
        base_url: Base URL for the ZwiftPower API
        results_dir: Directory for storing race results
        decimal_precision: Precision for decimal calculations
    """
    base_url: str = Field("https://www.zwiftpower.com", env="ZWIFT_API_BASE_URL")
    results_dir: Path = Field(Path("./results"), env="RESULTS_DIR")
    decimal_precision: int = Field(9, env="DECIMAL_PRECISION")

    class Config:
        env_file = ".env"
        env_file_encoding = "utf-8"
```

### **Data Models**

Contains Pydantic models for type-safe data validation and enumerated types for race categories and prime types.

```python
from enum import Enum, auto
from dataclasses import dataclass
from decimal import Decimal
from datetime import timedelta
from typing import List, Optional
from pydantic import BaseModel

class Category(str, Enum):
    """Race categories in Zwift competitions."""
    A = "a"
    B = "b"
    C = "c"
    D = "d"

class PrimeType(Enum):
    """Types of prime segments in races."""
    SPRINT = auto()
    KOM = auto()

@dataclass
class PrimeConfig:
    """Configuration for prime segments in each stage.

    Attributes:
        stage: Race stage number
        prime_type: Type of prime segment (Sprint or KOM)
        segment_ids: List of segment identifiers for the stage
    """
    stage: int
    prime_type: PrimeType
    segment_ids: List[str]

class RiderResult(BaseModel):
    """Model for individual rider results.

    Attributes:
        name: Rider's full name
        zwid: Zwift rider ID
        category: Race category
        gender: Rider's gender
        race_time: Decimal time in seconds
        points: Optional points earned
        team: Optional team affiliation
    """
    name: str
    zwid: str
    category: Category
    gender: str
    race_time: Decimal
    points: Optional[int] = None
    team: Optional[str] = None

    @property
    def display_time(self) -> str:
        """Format race time as HH:MM:SS."""
        return str(timedelta(seconds=float(self.race_time)))

class EventResults(BaseModel):
    """Model for overall event results.

    Attributes:
        results: List of individual rider results
        event_id: Unique identifier for the event
        stage: Race stage number
    """
    results: List[RiderResult]
    event_id: int
    stage: int
```

### **ZwiftAPIClient**

An asynchronous client for interacting with the ZwiftPower API, implementing proper resource management and error handling.

```python
import aiohttp
import logging
from typing import Optional

logger = logging.getLogger(__name__)

class ZwiftAPIClient:
    """Async client for interacting with ZwiftPower API.

    Implements async context management for proper resource handling
    and provides methods for fetching race results.
    """

    def __init__(self, settings: Settings):
        self.settings = settings
        self.session: Optional[aiohttp.ClientSession] = None
        self._headers = {
            "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36",
        }

    async def __aenter__(self) -> 'ZwiftAPIClient':
        """Initialize async HTTP session."""
        self.session = aiohttp.ClientSession(
            base_url=self.settings.base_url,
            headers=self._headers
        )
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        """Ensure proper cleanup of async resources."""
        if self.session:
            await self.session.close()

    async def get_event_results(self, event_id: int) -> Optional[EventResults]:
        """Fetch results for a specific event.

        Args:
            event_id: Unique identifier for the race event

        Returns:
            EventResults object if successful, None if request fails

        Raises:
            RuntimeError: If client is not initialized in context
        """
        if not self.session:
            raise RuntimeError("Client not initialized. Use async with context manager.")

        try:
            async with self.session.get(f"/cache3/results/{event_id}_zwift.json") as response:
                response.raise_for_status()
                data = await response.json()
                return EventResults(results=data["data"], event_id=event_id, stage=1)
        except aiohttp.ClientError as e:
            logger.error(f"Error fetching event {event_id}: {e}")
            return None
```

### **RaceProcessor**

Handles the processing of race results, calculating standings, and managing data persistence.

```python
import pandas as pd
from pathlib import Path
from typing import Dict

class RaceProcessor:
    """Process race results and calculate standings.

    Handles the computation of race results, general classification
    standings, and persistence of results to CSV files.
    """

    def __init__(self, settings: Settings):
        self.settings = settings
        self.results_dir = settings.results_dir
        self._gc_results: Dict[Category, pd.DataFrame] = {}

    def process_stage_results(self, results: EventResults) -> None:
        """Process results for a single stage.

        Converts results to DataFrame, calculates rankings by category,
        saves stage results, and updates GC standings.

        Args:
            results: EventResults object containing stage results
        """
        df = pd.DataFrame([r.dict() for r in results.results])

        for category in Category:
            category_results = df[df.category == category].copy()
            if not category_results.empty:
                category_results['rank'] = category_results['race_time'].rank()
                self._save_stage_results(
                    category_results,
                    results.stage,
                    category
                )
                self._update_gc_standings(category_results, category)

    def _save_stage_results(self, results: pd.DataFrame, stage: int, category: Category) -> None:
        """Save stage results to CSV.

        Args:
            results: DataFrame containing stage results
            stage: Stage number
            category: Race category
        """
        output_path = self.results_dir / f"stage_{stage}" / f"results_{category}.csv"
        output_path.parent.mkdir(parents=True, exist_ok=True)

        results.to_csv(output_path, index=False)
        logger.info(f"Saved stage {stage} results for category {category}")

    def _update_gc_standings(self, stage_results: pd.DataFrame, category: Category) -> None:
        """Update general classification standings with new stage results.

        Merges new stage results with existing GC standings and
        updates cumulative times.

        Args:
            stage_results: DataFrame containing new stage results
            category: Race category
        """
        if category not in self._gc_results:
            self._gc_results[category] = stage_results
        else:
            merged = pd.merge(
                self._gc_results[category],
                stage_results[['zwid', 'race_time']],
                on='zwid',
                how='outer',
                suffixes=('_gc', '_stage')
            )
            merged['race_time'] = merged['race_time_gc'].fillna(0) + merged['race_time_stage'].fillna(0)
            self._gc_results[category] = merged.drop(['race_time_gc', 'race_time_stage'], axis=1)

    def save_gc_results(self) -> None:
        """Save final GC standings to CSV.

        Calculates final rankings and saves results by category.
        """
        gc_dir = self.results_dir / "gc"
        gc_dir.mkdir(parents=True, exist_ok=True)

        for category, results in self._gc_results.items():
            final_results = results.sort_values('race_time')
            final_results['final_rank'] = range(1, len(final_results) + 1)

            output_path = gc_dir / f"gc_results_{category}.csv"
            final_results.to_csv(output_path, index=False)
            logger.info(f"Saved GC results for category {category}")
```

### **Main Execution**

Demonstrates the main execution flow of the application, including proper async handling and error management.

```python
async def main():
    """Main execution function.

    Sets up the application, processes race results, and
    generates final standings.
    """
    settings = Settings()

    async with ZwiftAPIClient(settings) as client:
        processor = RaceProcessor(settings)

        for event_id in [123, 456, 789]:  # Replace with actual event IDs
            if results := await client.get_event_results(event_id):
                processor.process_stage_results(results)

        processor.save_gc_results()

if __name__ == "__main__":
    asyncio.run(main())
```
