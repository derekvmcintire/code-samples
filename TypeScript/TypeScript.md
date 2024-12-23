# TypeScript Code Sample

**Author**: Derek McIntire  
**Date**: November 2024  
**Project Repository**: [GitHub - simple-fetch-ts](https://github.com/derekvmcintire/simple-fetch-ts)  
**NPM Package**: [simple-fetch-ts on NPM](https://www.npmjs.com/package/simple-fetch-ts)

---

## Overview

The `SimpleBuilder` class is a TypeScript utility for creating and executing HTTP requests using a builder-pattern-based API. This class simplifies HTTP requests by providing a fluent interface for chaining configuration methods, such as setting request bodies, headers, and query parameters, and supports common HTTP methods like GET, POST, PUT, PATCH, and DELETE.

The main goal of this utility is to provide an abstraction layer that allows developers to write HTTP requests more intuitively and maintainable.

### Key Features:

- Fluent interface using method chaining.
- Supports HTTP methods: GET, POST, PUT, PATCH, and DELETE.
- Allows configuration of headers, request body, and query parameters.
- Ensures that methods like GET do not accidentally include a request body.
- Includes error handling for invalid method-body combinations.

---

## Code

### **`SimpleBuilder` Class**

```typescript
import { InvalidMethodBodyError } from "../errors/request-body-error";
import { tsDelete } from "../methods/delete";
import { tsFetch } from "../methods/fetch";
import { tsPatch } from "../methods/patch";
import { tsPost } from "../methods/post";
import { tsPut } from "../methods/put";
import { QueryParams, SimpleResponse } from "../types";
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

  /**
   * Constructs a SimpleBuilder instance with a base URL and optional default headers.
   * @param url - The base URL for the request.
   * @param defaultHeaders - Default headers to include in every request.
   */
  constructor(url: string, defaultHeaders: HeadersInit = {}) {
    this.url = url;
    this.requestHeaders = defaultHeaders;
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
   * Checks if the given headers object is a plain record of key-value pairs.
   * @param headers - The headers object to validate.
   * @returns True if the headers object is a plain record, false otherwise.
   */
  private isHeadersRecord(
    headers: HeadersInit
  ): headers is Record<string, string> {
    return typeof headers === "object" && !(headers instanceof Headers);
  }

  /**
   * Ensures that the Content-Type header is set to 'application/json' if a request body exists.
   */
  private prepareHeaders(): void {
    if (
      this.isHeadersRecord(this.requestHeaders) &&
      !this.requestHeaders["Content-Type"] &&
      this.requestBody
    ) {
      this.requestHeaders["Content-Type"] = "application/json";
    }
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
    if (
      !["POST", "PUT", "PATCH", "DELETE"].includes(method) &&
      this.requestBody !== null
    ) {
      throw new InvalidMethodBodyError(method, this.url);
    }

    this.prepareHeaders();

    try {
      return await requestFn();
    } catch (error) {
      console.error(`Error with ${method} request to ${this.url}:`, error); // @TODO allow logger configuration
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

## Additional Notes

- The **`SimpleBuilder`** class supports all major HTTP methods and includes built-in error handling for invalid combinations of HTTP methods and request bodies.
- Query parameters are automatically serialized into the URL using the `serializeQueryParams` helper.
- The **`Content-Type`** header is automatically set to `application/json` if a request body is provided for methods like POST, PUT, or PATCH.

---
