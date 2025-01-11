# TypeScript Code Sample

**Author**: Derek McIntire  
**Date**: November 2024  
**Project Repository**: [derekvmcintire/receipt-processor](https://github.com/derekvmcintire/receipt-processor)

---

## Overview

This code sample demonstrates the testing of a ReceiptService in a TypeScript-based application. The ReceiptService is responsible for processing receipts and calculating reward points based on various rules such as retailer name, purchase total, and item details. It also retrieves the points associated with saved receipts. The tests ensure that the service correctly handles receipt validation, points calculation, and error scenarios, such as when invalid data is provided or when a receipt cannot be found.

The testing suite uses Jest as the testing framework, and mocks are used to simulate the repository and points calculator dependencies, allowing for focused unit tests. This approach ensures that each unit of the service logic is tested in isolation.

### Key Features:

- Unit Testing: The tests cover core functionalities of the ReceiptService, including receipt processing and points retrieval, ensuring the application behaves as expected under different scenarios.
- Error Handling: Tests verify that the service correctly throws errors when invalid data is passed or when the requested receipt cannot be found.
- Points Calculation: The tests verify that the correct number of points is calculated based on various factors, such as retailer name, total amount, and item details.
- Mocking: External dependencies like the InMemoryReceiptRepository and PointsCalculator are mocked to isolate the service's logic and focus on testing specific behavior.
- Comprehensive Coverage: The test suite includes both normal cases (valid data) and edge cases (invalid data, not found receipts), ensuring that the service can handle a wide range of input scenarios effectively.

---

## Code

### **`src/application/core/domain/services/receipt-service.test.ts`**

```typescript
import { ReceiptService } from "./receipt-service";
import { HTTPError } from "../../../../interface/errors/http-error";
import { PointsCalculator } from "../../utils/points-calculator";
import { InMemoryReceiptRepository } from "../../../../infrastructure/repositories/in-memory-receipt-repository";
import { ReceiptEntity } from "../entities/receipt";
import { mockReceipt } from "../../../../types/domain/receipt";

jest.mock("../../utils/points-calculator");

describe("ReceiptService", () => {
  let receiptService: ReceiptService;
  let mockRepository: jest.Mocked<InMemoryReceiptRepository>;
  let mockPointsCalculator: jest.Mocked<PointsCalculator>;

  const validReceiptData = mockReceipt;

  beforeEach(() => {
    // Mock repository methods
    mockRepository = {
      find: jest.fn(),
      save: jest.fn(),
    } as unknown as jest.Mocked<InMemoryReceiptRepository>;

    // Mock the getInstance method of the InMemoryReceiptRepository class
    jest
      .spyOn(InMemoryReceiptRepository, "getInstance")
      .mockReturnValue(mockRepository);

    // Mock PointsCalculator methods
    mockPointsCalculator =
      new PointsCalculator() as jest.Mocked<PointsCalculator>;
    mockPointsCalculator.calculatePoints.mockReturnValue(50);

    // Initialize ReceiptService with mocked dependencies
    receiptService = new ReceiptService(mockRepository, mockPointsCalculator);
  });

  describe("processReceipt", () => {
    it("should validate the receipt, calculate points, and save it", () => {
      const result = receiptService.processReceipt(validReceiptData);

      expect(result).toBeDefined(); // Receipt ID is generated
      expect(mockRepository.save).toHaveBeenCalledWith(
        expect.any(String), // Generated receipt ID
        expect.any(ReceiptEntity), // Validated ReceiptEntity
        50 // Calculated points
      );
      expect(mockPointsCalculator.calculatePoints).toHaveBeenCalledWith(
        expect.any(ReceiptEntity)
      );
    });

    it("should throw an error if receipt validation fails", () => {
      const invalidReceiptData = { ...validReceiptData, retailer: "" };

      expect(() => receiptService.processReceipt(invalidReceiptData)).toThrow(
        new HTTPError(
          "Invalid receipt data. Required fields: retailer, purchaseDate, purchaseTime, items, total.",
          400
        )
      );
      expect(mockPointsCalculator.calculatePoints).not.toHaveBeenCalled();
      expect(mockRepository.save).not.toHaveBeenCalled();
    });
  });

  describe("findReceiptPoints", () => {
    it("should return points when receipt is found", () => {
      const receiptId = "123";

      const receipt = new ReceiptEntity(validReceiptData);
      receipt.id = receiptId;

      // Simulate repository returning a saved receipt with points
      mockRepository.find.mockReturnValueOnce({
        ...receipt,
        points: 50,
      });

      const result = receiptService.findReceiptPoints(receiptId);

      expect(result).toEqual({ points: 50 });
      expect(mockRepository.find).toHaveBeenCalledWith(receiptId);
    });

    it("should throw 404 error when receipt is not found", () => {
      const receiptId = "nonexistent-id";

      // Simulate repository not finding the receipt
      mockRepository.find.mockReturnValueOnce(null);

      expect(() => receiptService.findReceiptPoints(receiptId)).toThrow(
        new HTTPError(
          `Unable to retrieve saved Receipt from id ${receiptId}`,
          404
        )
      );
    });
  });
});
```

### **`src/application/core/utils/points-calculator.test.ts`**

```typescript
import { mockReceipt } from "../../../types/domain/receipt";
import { PointsCalculator } from "./points-calculator";
import { getPointsCalculator } from "./points-calculator-factory";
import {
  calculateAlphaNumericCharactersPoints,
  calculateEveryTwoItemsPoints,
  calculateItemDescriptionLengthPoints,
  calculateMultipleOfPointTwentyFivePoints,
  calculateOddPurchaseDatePoints,
  calculatePurchaseTimePoints,
  calculateRoundDollarAmountPoints,
} from "./points-rules";

describe("Points Rules", () => {
  it("should calculate points for alphanumeric characters in retailer name", () => {
    const points = calculateAlphaNumericCharactersPoints.calculate(mockReceipt);
    expect(points).toBe(6); // "Target" has 6 total = 6
  });

  it("should calculate points for round dollar amounts", () => {
    const receipt = { ...mockReceipt, total: "50.00" };
    const points = calculateRoundDollarAmountPoints.calculate(receipt);
    expect(points).toBe(50);
  });

  it("should calculate points for totals that are multiples of 0.25", () => {
    const receipt = { ...mockReceipt, total: "35.25" };
    const points = calculateMultipleOfPointTwentyFivePoints.calculate(receipt);
    expect(points).toBe(25);
  });

  it("should calculate points for every two items", () => {
    const points = calculateEveryTwoItemsPoints.calculate(mockReceipt);
    expect(points).toBe(10); // 5 items, so 2 * 5 points
  });

  it("should calculate points for item description lengths that are multiples of 3", () => {
    const points = calculateItemDescriptionLengthPoints.calculate(mockReceipt);
    expect(points).toBe(6); // 6 total - 2 points for "Mountain Dew @ 6.49", 1 point for "Doritos Nacho Cheese @ 3.35" and 3 points for "Klarbrunn 12-PK 12 FL OZ @ 12.00"
  });

  it("should calculate points for odd purchase date", () => {
    const points = calculateOddPurchaseDatePoints.calculate(mockReceipt);
    expect(points).toBe(6); // Day 1 is odd
  });

  it("should calculate points for purchase time between 2:00 PM and 4:00 PM", () => {
    const receipt = { ...mockReceipt, purchaseTime: "14:30" };
    const points = calculatePurchaseTimePoints.calculate(receipt);
    expect(points).toBe(10);
  });

  it("should calculate zero points for purchase time outside 2:00 PM - 4:00 PM", () => {
    const receipt = { ...mockReceipt, purchaseTime: "16:01" };
    const points = calculatePurchaseTimePoints.calculate(receipt);
    expect(points).toBe(0);
  });
});

describe("Points Calculator", () => {
  let calculator: PointsCalculator;

  beforeEach(() => {
    calculator = new PointsCalculator();
    calculator.registerRule(calculateAlphaNumericCharactersPoints);
    calculator.registerRule(calculateRoundDollarAmountPoints);
    calculator.registerRule(calculateMultipleOfPointTwentyFivePoints);
    calculator.registerRule(calculateEveryTwoItemsPoints);
    calculator.registerRule(calculateItemDescriptionLengthPoints);
    calculator.registerRule(calculateOddPurchaseDatePoints);
    calculator.registerRule(calculatePurchaseTimePoints);
  });

  it("should calculate total points for a receipt", () => {
    const points = calculator.calculatePoints(mockReceipt);
    expect(points).toBe(28); // Sum of all rules for mockReceipt
  });
});

describe("Points Factory", () => {
  it("should create a PointsCalculator with all rules registered", () => {
    const calculator = getPointsCalculator();
    const points = calculator.calculatePoints(mockReceipt);
    expect(points).toBe(28); // Verify the factory registers all rules
  });
});
```
