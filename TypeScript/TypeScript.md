# TypeScript Code Sample

**Author**: Derek McIntire  
**Date**: November 2024  
**Project Repository**: [GitHub - simple-fetch-ts](https://github.com/derekvmcintire/simple-fetch-ts)  
**NPM Package**: [simple-fetch-ts on NPM](https://www.npmjs.com/package/simple-fetch-ts)

---

## Overview

The `simple-fetch-ts` package provides a fluent, builder-pattern-based interface for making HTTP requests. This TypeScript utility simplifies the process of creating and sending requests, with built-in error handling, customizable headers, query parameters, and request bodies. The library supports the following HTTP methods:

- **GET**: For retrieving data.
- **POST**: For submitting data.
- **PUT**: For updating data.
- **PATCH**: For partially updating data.
- **DELETE**: For deleting data.

The primary goal of this package is to provide a lightweight, intuitive abstraction for working with HTTP requests, while maintaining flexibility and type safety, making it easier for developers to build and manage network interactions in TypeScript.

### Key Features:

- **Fluent Interface**: Chain configuration methods for headers, body, and query parameters.
- **Type Safety**: Built-in TypeScript support for strongly-typed request bodies and responses.
- **Automatic Error Handling**: Catches common HTTP errors and method-body mismatches.
- **Customizable Query Parameters**: Allows serialization and validation of query parameters.

---

## Code

### **`simple` Factory Function**

This factory function simplifies the creation of a `SimpleBuilder` instance with URL validation and optional custom logging.

```typescript
import { SimpleBuilder } from "../builder";
import { InvalidURLError } from "../errors/url-validation-error";
import { SimpleLogger } from "../types";
import { isValidURL } from "../utility/url-helpers";

/**
 * Factory function to create an instance of SimpleBuilder.
 *
 * This function validates the provided URL and throws an error if it's invalid.
 * It then returns a new instance of SimpleBuilder, optionally with a custom logger.
 *
 * @param {string} url - The base URL for the HTTP requests.
 * @param {SimpleLogger} [logger=console.error] - Optional custom logger function for logging errors.
 * @returns {SimpleBuilder} A new instance of SimpleBuilder.
 * @throws {InvalidURLError} If the provided URL is invalid.
 */
export const simple = (url: string, logger?: SimpleLogger): SimpleBuilder => {
  if (!isValidURL(url)) {
    throw new InvalidURLError(url);
  }

  return new SimpleBuilder({ url, logger });
};
```

---

### **`SimpleBuilder` Class**

The `SimpleBuilder` class is the core component for configuring and sending HTTP requests. It provides methods for setting the request body, headers, and query parameters, while also ensuring proper error handling and validation of the request configuration.

- The **`SimpleBuilder`** class supports all major HTTP methods and includes built-in error handling for invalid combinations of HTTP methods and request bodies.
- Query parameters are automatically serialized into the URL using the `serializeQueryParams` helper.
- The **`Content-Type`** header is automatically set to `application/json` if a request body is provided for methods like POST, PUT, or PATCH.

```typescript
import { InvalidMethodBodyError } from "../errors/request-body-error";
import { SimpleFetchRequestError } from "../errors/request-error";
import { tsDelete } from "../methods/delete";
import { tsFetch } from "../methods/fetch";
import { tsPatch } from "../methods/patch";
import { tsPost } from "../methods/post";
import { tsPut } from "../methods/put";
import { QueryParams, SimpleLogger, SimpleResponse } from "../types";
import { serializeQueryParams } from "../utility/url-helpers";

/**
 * A class for creating and executing HTTP requests with a fluent, builder-pattern-based
 * abstraction. SimpleBuilder provides support for common HTTP methods like GET, POST, PUT, PATCH,
 * and DELETE, and allows configuration of headers, query parameters, and request body.
 */
export class SimpleBuilder {
  private url: string;
  private requestBody: unknown = null;
  private requestHeaders: HeadersInit = {};
  private requestParams: string = "";
  private logger: SimpleLogger;

  /**
   * Constructs a SimpleBuilder instance with a base URL and optional default headers.
   * @param options - The options for configuring the SimpleBuilder instance.
   * @param defaultHeaders - Default headers to include in every request.
   */
  constructor({
    url,
    logger = console.error,
    defaultHeaders = {},
  }: SimpleBuilderOptions) {
    this.url = url;
    this.requestHeaders = defaultHeaders;
    this.logger = logger;
  }

  /**
   * Sets the request body for the HTTP request.
   * @template T - The type of the body.
   * @param body - The request payload.
   * @returns The current SimpleBuilder instance for method chaining.
   */
  body<T>(body: T): SimpleBuilder {
    this.requestBody = body;
    return this;
  }

  /**
   * Adds or updates headers for the HTTP request.
   * @param requestHeaders - The headers to merge with existing headers.
   * @returns The current SimpleBuilder instance for method chaining.
   */
  headers(requestHeaders: HeadersInit): this {
    this.requestHeaders = { ...this.requestHeaders, ...requestHeaders };
    return this;
  }

  /**
   * Serializes and appends query parameters to the URL.
   * @param requestParams - The query parameters as an object.
   * @param lowerCaseKeys - Whether to convert all parameter keys to lowercase.
   * @returns The current SimpleBuilder instance for method chaining.
   */
  params(requestParams: QueryParams, lowerCaseKeys: boolean = false): this {
    this.requestParams = serializeQueryParams(requestParams, lowerCaseKeys);
    return this;
  }

  /**
   * Builds the final URL by appending query parameters if they exist.
   * @returns The constructed URL with query parameters.
   */
  private buildUrl(): string {
    return this.requestParams ? `${this.url}?${this.requestParams}` : this.url;
  }

  /**
   * Executes the provided request function and handles errors.
   * Validates that the request body is only used with applicable HTTP methods.
   * @param requestFn - The function to execute the HTTP request.
   * @param method - The HTTP method being used.
   * @returns A promise resolving to the response of the HTTP request.
   * @throws An error if the method is invalid for requests with a body.
   */
  private async handleRequest<T>(
    requestFn: () => Promise<SimpleResponse<T>>,
    method: "GET" | "POST" | "PUT" | "PATCH" | "DELETE"
  ): Promise<SimpleResponse<T>> {
    const { logger, url } = this;

    if (
      !["POST", "PUT", "PATCH", "DELETE"].includes(method) &&
      this.requestBody !== null
    ) {
      const error = new InvalidMethodBodyError(method, url);
      logger(error.message, error);
      throw error;
    }

    this.prepareHeaders();

    try {
      return await requestFn();
    } catch (error) {
      if (error instanceof SimpleFetchRequestError) {
        logger(
          `Error with ${method} request to ${url}: Status ${error.status} - ${error.statusText}`,
          error
        );
      } else if (error instanceof Error) {
        logger(`Error with ${method} request to ${url}:`, error);
      } else {
        const unknownError = new Error("Unknown error type encountered.");
        logger(`Error with ${method} request to ${url}:`, unknownError);
      }
      throw error;
    }
  }

  /**
   * Executes a GET request.
   * @template T - The type of the expected response data.
   * @returns A promise resolving to the response of the GET request.
   */
  async fetch<T>(): Promise<SimpleResponse<T>> {
    const fullUrl = this.buildUrl();
    return this.handleRequest(
      () => tsFetch<T>(fullUrl, this.requestHeaders),
      "GET"
    );
  }

  /**
   * Executes a POST request with the configured body and headers.
   * @template T - The type of the expected response data.
   * @returns A promise resolving to the response of the POST request.
   */
  async post<T>(): Promise<SimpleResponse<T>> {
    return this.handleRequest(
      () => tsPost<T>(this.buildUrl(), this.requestBody, this.requestHeaders),
      "POST"
    );
  }

  /**
   * Executes a PUT request with the configured body and headers.
   * @template T - The type of the expected response data.
   * @returns A promise resolving to the response of the PUT request.
   */
  async put<T>(): Promise<SimpleResponse<T>> {
    return this.handleRequest(
      () => tsPut<T>(this.buildUrl(), this.requestBody, this.requestHeaders),
      "PUT"
    );
  }

  /**
   * Executes a PATCH request with the configured body and headers.
   * @template T - The type of the expected response data.
   * @returns A promise resolving to the response of the PATCH request.
   */
  async patch<T>(): Promise<SimpleResponse<T>> {
    return this.handleRequest(
      () => tsPatch<T>(this.buildUrl(), this.requestBody, this.requestHeaders),
      "PATCH"
    );
  }

  /**
   * Executes a DELETE request with the configured headers.
   * @template T - The type of the expected response data.
   * @returns A promise resolving to the response of the DELETE request.
   */
  async delete<T>(): Promise<SimpleResponse<T>> {
    return this.handleRequest(
      () => tsDelete<T>(this.buildUrl(), this.requestHeaders),
      "DELETE"
    );
  }
}
```

---

### **`tsPost` Method**

The tsPost function implements the actual POST request by serializing the request body if needed and returning a type-safe response.

```typescript
import { SimpleFetchRequestError } from "../../errors/request-error";
import { SimpleResponse } from "../../types";
import { getContentType } from "../../utility/get-content-type";

/**
 * Performs a typed POST request to the specified URL.
 *
 * If the `Content-Type` is set to `application/json` and the body is an object, the body
 * is automatically stringified to JSON format.
 *
 * @template T - The type of the expected response data.
 * @param url - The URL to send the request to.
 * @param requestBody - The data to be sent as the request body.
 * @param requestHeaders - Optional headers to include with the request.
 * @returns A promise that resolves with a SimpleResponse object.
 * @throws Will throw an error if the fetch fails or the response status is not OK.
 */
export const tsPost = async <T>(
  url: string,
  requestBody: any,
  requestHeaders: HeadersInit = {}
): Promise<SimpleResponse<T>> => {
  const contentType = getContentType(requestHeaders).toLowerCase();

  // Automatically stringify the body if Content-Type is JSON and body is an object
  const body =
    contentType === "application/json" && typeof requestBody === "object"
      ? JSON.stringify(requestBody)
      : requestBody;

  try {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        "Content-Type": contentType,
        ...requestHeaders,
      },
      body,
    });

    if (!response.ok) {
      throw new SimpleFetchRequestError(response);
    }

    return {
      status: response.status,
      statusText: response.statusText,
      data: await response.json(),
    };
  } catch (error) {
    throw new SimpleFetchRequestError(error);
  }
};
```
