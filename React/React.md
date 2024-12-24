# React Code Sample

**Author**: Derek McIntire  
**Date**: November 2024  
**Project Repository**: [GitHub - acs-next](https://github.com/derekvmcintire/acs-next)

---

## Overview

This code sample demonstrates the implementation of two React components:

1. **`SearchAutoComplete`**: A reusable search input component powered by Mantine’s `Autocomplete` and React icons.
2. **`RaceSearch`**: A component that integrates `SearchAutoComplete` to search for races, utilizing React Query to fetch race data and manage state with debounce functionality.

These components showcase my ability to work with modern React patterns, TypeScript, and the Mantine component library, as well as best practices for clean, maintainable code.

---

## Code

### 1. **`SearchAutoComplete` Component**

This is a simple, reusable autocomplete component that allows users to search for a race. It supports customization for placeholder text, search result limits, and event handling.

```typescript
"use client";

import React from "react";
import { Autocomplete } from "@mantine/core";
import { GoSearch } from "react-icons/go";
import classes from "./search-autocomplete.module.css";

const icon = <GoSearch aria-hidden="true" />;

export type SearchOption = {
  value: string;
  label: string;
};

interface SearchAutoCompleteProps {
  leftSection?: React.ReactNode;
  leftSectionPointerEvents?: React.CSSProperties["pointerEvents"];
  placeholder?: string;
  data: SearchOption[];
  limit: number;
  value: string;
  onChange: (value: string) => void;
  onOptionSubmit: (value: string) => void;
}

const SearchAutoComplete: React.FC<SearchAutoCompleteProps> = React.memo(
  function SearchAutoComplete({
    leftSectionPointerEvents = "none",
    placeholder = "Search",
    data,
    limit,
    value,
    onChange,
    onOptionSubmit,
  }: SearchAutoCompleteProps) {
    return (
      <Autocomplete
        classNames={{
          input: classes.searchAutocomplete,
        }}
        leftSectionPointerEvents={leftSectionPointerEvents}
        leftSection={icon}
        placeholder={placeholder}
        data={data}
        limit={limit}
        value={value}
        onChange={onChange}
        onOptionSubmit={onOptionSubmit}
      />
    );
  }
);

export default SearchAutoComplete;
```

#### Key Features:

- **Customizable placeholder** and **left section** (for the search icon).
- **Debounce** for handling input changes efficiently.
- **TypeScript** for type safety.

---

### 2. **`RaceSearch` Component**

This component uses the `SearchAutoComplete` component to search for races. It integrates **React Query** for asynchronous data fetching and supports error handling, caching, and a debounced input for optimal performance.

```typescript
"use client";

import React from "react";
import { Container } from "@mantine/core";
import { useQuery } from "@tanstack/react-query";
import { fetchRaces } from "@/src/_api/get/races/fetch-races";
import { useUploaderContext } from "@/src/_contexts/Uploader/UploaderContext";
import useDebounce from "@/src/_hooks/use-debounce";
import { getFormattedYearString, yearTrunc } from "@/src/_utility/date-helpers";
import { IGetRacesResponse } from "../../../_api/types";
import SearchAutoComplete from "../../shared/SearchAutocomplete";
import SectionLabel from "../../ui/SectionLabel";

type SearchOption = {
  value: string;
  label: string;
};

type RaceSearchProps = {
  setError: (message: string) => void;
};

export default function RaceSearch({ setError }: RaceSearchProps) {
  const [searchValue, setSearchValue] = React.useState<string>("");
  const debouncedSearchValue = useDebounce(searchValue, 300); // 300ms debounce

  const { setSelectedRace } = useUploaderContext();

  const {
    data: searchResponse,
    isError,
    error,
  } = useQuery<IGetRacesResponse | undefined>({
    queryKey: ["getRaces", debouncedSearchValue],
    queryFn: () => fetchRaces({ name: debouncedSearchValue }),
    enabled: debouncedSearchValue.length > 0,
    staleTime: 5 * 60 * 1000, // caches data from requests for 5 minutes
  });

  const hasAvailableRaces =
    searchResponse?.races && searchResponse.races.length > 0;
  const searchRaces = searchResponse?.races || [];

  React.useEffect(() => {
    if (isError && error) {
      setError(String(error));
    } else if (!isError && !error) {
      setError("");
    }
  }, [isError, error, setError]);

  const options: SearchOption[] = React.useMemo(() => {
    if (!hasAvailableRaces) return [];
    return searchRaces.map((race) => {
      const year = getFormattedYearString(new Date(race.startDate));
      return {
        value: String(race.id),
        label: `${race.event.name} ${yearTrunc(Number(year), true)}`,
      };
    });
  }, [searchRaces]);

  const findRaceById = React.useCallback(
    (id: number) => {
      return searchRaces.find((race) => race.id === id);
    },
    [searchRaces]
  );

  const handleChange = React.useCallback((input: string) => {
    setSearchValue(input);
  }, []);

  const handleOptionSubmit = (option: string) => {
    const fullRace = findRaceById(Number(option));
    setSelectedRace(fullRace);
  };

  return (
    <Container mb="36px">
      <SectionLabel text="Select a Race" />
      <SearchAutoComplete
        data={options}
        limit={15}
        value={searchValue}
        onChange={handleChange}
        onOptionSubmit={handleOptionSubmit}
        aria-label="Search for a race"
      />
    </Container>
  );
}
```

#### Key Features:

- **React Query** for fetching and caching race data.
- **Debounced input** to optimize API calls.
- **Custom error handling** that integrates with the parent component’s state.

---

### 3. **Uploader Context**

The Uploader Context provides a centralized state management system for handling the process of uploading and managing race-related data.

```typescript
"use client";

import React, {
  createContext,
  ReactNode,
  useContext,
  useState,
  useMemo,
} from "react";
import { GetCategoriesResponse } from "@/src/_api/get/categories/fetch-categories-response-type";
import { IGetRacesResponse } from "../../_api/types";

export interface IUploaderContext {
  selectedRace?: IGetRacesResponse | undefined; // Explicit type for race
  setSelectedRace: (selectedRace: IGetRacesResponse | undefined) => void;
  categoryOptions: GetCategoriesResponse[];
  setCategoryOptions: (categoryOptions: GetCategoriesResponse[]) => void;
  isLoading: boolean;
  setIsLoading: (isLoading: boolean) => void;
  errors: string[];
  setErrors: (errors: string[]) => void;
  successMessage: string;
  setSuccessMessage: (successMessage: string) => void;
}

export const defaultUploaderContextValue: IUploaderContext = {
  selectedRace: undefined,
  setSelectedRace: () => {},
  categoryOptions: [],
  setCategoryOptions: () => {},
  isLoading: false,
  setIsLoading: () => {},
  errors: [],
  setErrors: () => {},
  successMessage: "",
  setSuccessMessage: () => {},
};

const UploaderContext = createContext<IUploaderContext>(
  defaultUploaderContextValue
);

interface UploaderContextProviderProps {
  children: ReactNode;
  initialValue?: IUploaderContext;
}

export const UploaderContextProvider: React.FC<
  UploaderContextProviderProps
> = ({ children, initialValue = defaultUploaderContextValue }) => {
  const [selectedRace, setSelectedRace] = useState<
    IGetRacesResponse | undefined
  >(initialValue.selectedRace);
  const [categoryOptions, setCategoryOptions] = useState<
    GetCategoriesResponse[]
  >(initialValue.categoryOptions);
  const [isLoading, setIsLoading] = useState<boolean>(false);
  const [errors, setErrors] = useState<string[]>(initialValue.errors);
  const [successMessage, setSuccessMessage] = useState<string>(
    initialValue.successMessage
  );

  const contextValue = useMemo(
    () => ({
      selectedRace,
      setSelectedRace,
      categoryOptions,
      setCategoryOptions,
      isLoading,
      setIsLoading,
      errors,
      setErrors,
      successMessage,
      setSuccessMessage,
    }),
    [selectedRace, categoryOptions, isLoading, errors, successMessage]
  );

  return (
    <UploaderContext.Provider value={contextValue}>
      {children}
    </UploaderContext.Provider>
  );
};

export const useUploaderContext = (): IUploaderContext => {
  const context = useContext(UploaderContext);
  if (!context) {
    throw new Error(
      "useUploaderContext must be used within an UploaderContextProvider"
    );
  }
  return context;
};
```

#### Key Features:

- **Centralized State Management** for race selection, category options, loading states, and errors in a user-friendly and type-safe manner.
- **UploaderContextProvider** wraps components that need access to these states.
- **useUploaderContext** custom hook for providing access to the UploaderContext.

---

## Additional Notes

- **Mantine UI**: This project utilizes Mantine, a modern React UI library that provides accessible and customizable components like the `Autocomplete` used in this example.
- **React Query**: Used for data fetching and caching, ensuring that requests are efficient and responses are properly cached.
- **Debounce Hook**: Custom hook to delay the search input and optimize API calls, preventing unnecessary requests while typing.

---
