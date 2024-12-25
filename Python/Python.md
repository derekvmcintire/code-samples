# Python Code Sample

**Author**: Derek McIntire  
**Date**: January 2022  
**Project Repository**: [derekvmcintire/veganuary](https://github.com/derekvmcintire/veganuary)

## Overview

This codebase demonstrates a system for interacting with ZwiftPower APIs, processing race results, and managing general classifications (GC) for cycling competitions. It covers making HTTP requests, handling CSV data, formatting race times, and maintaining configuration constants for race categories and prime results.

## Key Concepts

- **Modular Design**: Separation of concerns across multiple files and utilities.
- **API Integration**: Using HTTP requests to interact with ZwiftPower endpoints.
- **Data Processing**: Managing race results, GC rankings, and prime results with Panda.
- **Type Safety**: Using Pydantic models to validate API response structures.
- **File Operations**: Reading and writing structured data to CSV files with pathlib and csv.
- **Time Handling**: Formatting and manipulating race times.

---

## Code

### **`ZPRequests` Class**

A class that handles interaction with ZwiftPower APIs. It provides methods for loading race results and prime (sprint/KOM) results for a given event. It uses an injected `HttpClient` for making HTTP requests and integrates with Pydantic models for type validation of API responses.

```python
from typing import Any, Dict, List, Optional
from utils.http_client import HttpClient
from api.models import EventResults, PrimeResults


class ZPRequests:
    BASE_URL = "https://www.zwiftpower.com"
    RESULT_ENDPOINT = "/cache3/results/{id}_zwift.json"
    PRIME_ENDPOINT = "/api3.php?do=event_sprints&zid={id}"

    def __init__(self, http_client: HttpClient, event_id: Optional[int] = None):
        self.http_client = http_client
        self.event_id = event_id

    def load_event_results(self) -> Optional[EventResults]:
        """
        Load event
```

---

### **`HttpClient` Class**

A utility class for performing HTTP requests. It provides methods for making `GET` and `OPTIONS` requests and handles response parsing, error reporting, and custom headers to mimic a browser client.

```python
import requests
from typing import Any, Dict, Optional


class HttpClient:
    HEADERS = {
        "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/87.0.4280.88 Safari/537.36",
    }

    @staticmethod
    def get_json(url: str) -> Optional[Dict[str, Any]]:
        """
        Perform a GET request and return the response as JSON.
        """
        try:
            response = requests.get(url, headers=HttpClient.HEADERS)
            response.raise_for_status()
            return response.json()
        except requests.RequestException as e:
            print(f"HTTP error on GET {url}: {e}")
        except ValueError as e:
            print(f"JSON parsing error on GET {url}: {e}")
        return None

    @staticmethod
    def options_json(url: str) -> Optional[Dict[str, Any]]:
        """
        Perform an OPTIONS request and return the response as JSON.
        """
        try:
            response = requests.options(url, headers=HttpClient.HEADERS)
            response.raise_for_status()
            return response.json()
        except requests.RequestException as e:
            print(f"HTTP error on OPTIONS {url}: {e}")
        except ValueError as e:
            print(f"JSON parsing error on OPTIONS {url}: {e}")
        return None
```

---

### **Domain Models**

Contains Pydantic models used to validate and structure API responses. These models represent individual rider results, event results, and prime (sprint/KOM) results in a type-safe way.

```python
from typing import Any, Dict, List, Optional
from pydantic import BaseModel


class RiderResult(BaseModel):
    name: str
    category: str
    gender: str
    time: Optional[str] = None
    points: Optional[int] = None
    team: Optional[str] = None


class EventResults(BaseModel):
    results: List[RiderResult]


class PrimeResults(BaseModel):
    primes: Dict[str, List[RiderResult]]
```

---

### **`OverallStandingsModel` Class**

A class responsible for calculating and ranking General Classification (GC) results across multiple stages of a cycling competition. It processes stage results from CSV files, calculates cumulative GC times, ranks riders, and saves results back to CSV. It also handles sprint and KOM (King of the Mountain) results for additional classifications.

```python
from typing import List, Dict, Any
from decimal import Decimal, getcontext
from pathlib import Path

import pandas as pd
from utils.csv_utils import read_csv_as_dicts, write_dicts_to_csv
from utils.time_utils import format_race_time
from config.constants import CATEGORY_SHAPE, CATEGORIES, KOM_TYPE, SPRINT_TYPE, PRIME_IDS
from models.riders_model import RidersCollection

getcontext().prec = 9  # Set Decimal precision globally


class OverallStandingsModel:
    def __init__(self, last_stage: int):
        self.last_stage = last_stage
        self.stage_keys = list(range(1, last_stage + 1))
        self.sprint_data: Dict[str, Any] = {}
        self.kom_data: Dict[str, Any] = {}
        self.gc_results: Dict[str, List[Dict[str, Any]]] = {}
        self.ranked_gc = CATEGORY_SHAPE.copy()
        self.ranked_wgc = CATEGORY_SHAPE.copy()

        self.riders_collection = RidersCollection()
        self.stages_results: Dict[str, Dict[str, List[Dict[str, Any]]]] = {}

        self._load_stages_results()
        self._calculate_and_rank_gc()

    def _load_stages_results(self) -> None:
        for stage in self.stage_keys:
            stage_results = {}
            for category in CATEGORIES:
                file_path = f"./results/stage_{stage}/stage_{stage}_results_{category}.csv"
                stage_results[category] = read_csv_as_dicts(file_path)
            self.stages_results[str(stage)] = stage_results

    def _load_prime_results(self, prime_type: str, gender: str) -> Dict[str, Any]:
        prime_results = {}
        for stage in self.stage_keys:
            stage_results = {}
            for prime in PRIME_IDS.get(str(stage), {}).get(prime_type, []):
                prime_key = f"{prime_type}_{prime}"
                stage_results[prime_key] = {}
                for category in CATEGORIES:
                    file_path = f"./results/stage_{stage}/{gender}_prime_results_{category}_{prime}.csv"
                    stage_results[prime_key][category] = read_csv_as_dicts(file_path)
            prime_results[str(stage)] = stage_results
        return prime_results

    def _calculate_gc_times(self) -> None:
        for stage in self.stage_keys[1:]:
            for category, results in self.stages_results[str(stage)].items():
                for result in results:
                    current_gc_time = Decimal(self._get_rider_gc_time(result))
                    updated_gc_time = Decimal(result["race_time"]) + current_gc_time
                    result["race_time"] = updated_gc_time
                    result["display_race_time"] = format_race_time(updated_gc_time)

    def _get_rider_gc_time(self, rider: Dict[str, Any]) -> Decimal:
        for result in self.gc_results.get(rider["category"], []):
            if result["zwid"] == rider["zwid"]:
                return Decimal(result["race_time"])
        return Decimal(0)

    def _rank_gc_results(self) -> None:
        for category, results in self.gc_results.items():
            ranked = sorted(results, key=lambda x: x["race_time"])
            self.ranked_gc[category] = ranked
            self.ranked_wgc[category] = [r for r in ranked if r["gender"] == "2"]

    def _calculate_and_rank_gc(self) -> None:
        self.gc_results = self.stages_results["1"].copy()
        self._calculate_gc_times()
        self._rank_gc_results()

    def save_gc_results(self) -> None:
        for category, data in self.ranked_gc.items():
            file_path = f"./results/gc/cat_{category}_a_results.csv"
            write_dicts_to_csv(file_path, data)

        for category, data in self.ranked_wgc.items():
            file_path = f"./results/gc/cat_{category}_w_results.csv"
            write_dicts_to_csv(file_path, data)
```

---

### **CSV Utility Functions**

Provides utility functions for reading and writing structured data in CSV format. These utilities streamline file operations for managing race results and rankings.

```python
import csv
from typing import List, Dict


def read_csv_as_dicts(file_path: str) -> List[Dict[str, str]]:
    try:
        with open(file_path, "r") as file:
            return list(csv.DictReader(file))
    except FileNotFoundError:
        print(f"Error: File not found at {file_path}")
        return []


def write_dicts_to_csv(file_path: str, data: List[Dict[str, str]]) -> None:
    if not data:
        return

    with open(file_path, "w", newline="") as file:
        writer = csv.DictWriter(file, fieldnames=data[0].keys())
        writer.writeheader()
        writer.writerows(data)
```

---

### **Time Utility FUnction**

A utility function to format race times (in seconds) into a human-readable `hh:mm:ss` format. This is used for display purposes in the processed GC results.

```python
from datetime import timedelta


def format_race_time(race_time: float) -> str:
    return str(timedelta(seconds=float(race_time)))
```

---

### **Config Constants**

Defines constants for race categories (`A`, `B`, `C`, `D`), sprint/KOM types, and prime result IDs for different stages. These constants standardize configuration across the codebase and ensure consistency.

```python
KOM_TYPE = "kom"
SPRINT_TYPE = "sprint"

CATEGORIES = ['a', 'b', 'c', 'd']
CATEGORY_SHAPE = {
    "a": [],
    "b": [],
    "c": [],
    "d": []
}
PRIME_CATEGORY_SHAPE = {
    "a": {},
    "b": {},
    "c": {},
    "d": {}
}
WINNING_TIMES_SHAPE = {
    "a": 0,
    "b": 0,
    "c": 0,
    "d": 0
}
PRIME_CATEGORY_SHAPE = {
    "a": {},
    "b": {},
    "c": {},
    "d": {}
}
SINGLE_POINTS = [12, 10, 8, 7, 6, 5, 4, 3, 2, 1]
DOUBLE_POINTS = [20, 15, 10, 8, 6, 5, 4, 3, 2, 1]
SPRINT_TYPE = 1
KOM_TYPE = 2
PRIME_IDS = {
    '1': {
        KOM_TYPE: ['48'],
        SPRINT_TYPE: []
    },
    '2': {
        KOM_TYPE: ['54'],
        SPRINT_TYPE: ['59', '61', '62']
    },
    '3': {
        KOM_TYPE: ['20'],
        SPRINT_TYPE: ['21']
    },
    '4': {
        KOM_TYPE: ['2', '17'],
        SPRINT_TYPE: ['3']
    }
}
```
