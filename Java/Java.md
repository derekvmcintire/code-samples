# Java Code Sample

**Author**: Derek McIntire  
**Date**: November 2024  
**Project Repository**: [derekvmcintire/java-gql-receipt-processor](https://github.com/derekvmcintire/java-gql-receipt-processor)

## Overview

This code sample demonstrates a receipt processing system built using the Java Spring Framework. The system calculates reward points based on various rules applied to the data provided by a receipt. Each rule is represented by a separate class, making it easy to extend the system by adding new rules without modifying existing code. This design utilizes Spring's Dependency Injection to register and manage rules dynamically.

The core of the logic is the PointsCalculator class, which leverages the RuleRegistry to apply all registered rules to a receipt and compute the total points.

### Core Concepts:

- **Rule-based System**: Each point calculation rule is implemented as a separate class that adheres to the `Rule` interface.
- **Spring Dependency Injection**: The rules are automatically injected into the `RuleRegistry` and are easily extendable.
- **Custom Receipt Processing**: The `PointsCalculator` uses the rules to calculate points based on the given receipt's details (e.g., store name, total amount, item descriptions, purchase time).

---

## Code

### **`PointsCalculator` Class**

The `PointsCalculator` class is responsible for calculating the total points for a given receipt. It does this by applying all registered rules through the `RuleRegistry`.

```java
package ReceiptProcessor.application.utility;

import org.springframework.stereotype.Component;

import ReceiptProcessor.api.dto.AddReceiptInput;

/**
 * PointsCalculator is responsible for calculating points based on a given
 * receipt.
 *
 * It uses a list of rules to determine how many points to assign to a receipt.
 * These rules are retrieved from the RuleRegistry, which holds all Rule beans
 * (i.e., classes that implement the Rule interface and are annotated
 * with @Component).
 *
 * Spring will automatically inject the RuleRegistry bean, which contains all
 * the rules,
 * so new rules can be added or modified without needing to change this class.
 */
@Component // Marks this class as a Spring-managed component for dependency injection.
public class PointsCalculator {

  // The RuleRegistry is injected automatically by Spring. It provides access to
  // all Rule beans.
  private final RuleRegistry ruleRegistry;

  /**
   * Constructor for PointsCalculator.
   *
   * @param ruleRegistry An instance of RuleRegistry, automatically injected by
   *                     Spring.
   *                     It holds a list of all the Rule implementations.
   */
  public PointsCalculator(RuleRegistry ruleRegistry) {
    this.ruleRegistry = ruleRegistry; // Assign the injected RuleRegistry to the private field.
  }

  /**
   * Calculates the total points for the given receipt by applying all registered
   * rules.
   *
   * This method iterates through all rules in the RuleRegistry and applies each
   * one
   * to the provided receipt. The results from all rules are summed to get the
   * total points.
   *
   * @param receipt The receipt for which the points are calculated.
   * @return The total points calculated based on the rules.
   */
  public int calculatePoints(AddReceiptInput receipt) {
    return ruleRegistry.getRules() // Retrieves the list of rules from RuleRegistry.
        .stream() // Converts the list to a Stream to apply operations on each rule.
        .mapToInt(rule -> rule.calculate(receipt)) // For each rule, apply the calculate method and get the result
        .sum(); // Sums up the results of all rules to get the total points.
  }
}
```

### **`RuleRegistry` Class**

The `RuleRegistry` class is responsible for holding a list of all the `Rule` implementations. It automatically injects all `@Component`-annotated classes that implement the `Rule` interface, making it easy to register and apply rules.

```java
package ReceiptProcessor.application.utility;

import java.util.List;

import org.springframework.stereotype.Component;

/**
 * RuleRegistry is a Spring-managed component that holds a list of all the Rule
 * implementations.
 *
 * The rules are automatically injected by Spring when the application starts,
 * using
 * Dependency Injection (DI) to populate the 'rules' list with all beans that
 * implement
 * the 'Rule' interface. This allows you to easily extend or add new rules
 * without modifying
 * this class.
 *
 * Each Rule implementation is expected to be annotated with @Component, and
 * Spring will
 * automatically discover and register them during the component scanning
 * process.
 *
 * This class exposes the list of rules to be used in other components, like
 * PointsCalculator.
 */
@Component // Marks this class as a Spring-managed bean to be injected into other
           // components.
public class RuleRegistry {

  // A list of all Rule beans, automatically populated by Spring with any classes
  // annotated with @Component and implementing the Rule interface.
  private final List<Rule> rules;

  /**
   * Constructor for RuleRegistry.
   *
   * @param rules A list of Rule beans, automatically injected by Spring.
   *              Spring scans the application for any beans that implement
   *              the Rule interface and automatically populates this list.
   */
  public RuleRegistry(List<Rule> rules) {
    this.rules = rules; // Assigns the injected list of rules to the private field.
  }

  /**
   * Returns the list of Rule beans.
   *
   * @return List of all Rule beans that were injected by Spring.
   */
  public List<Rule> getRules() {
    return rules;
  }
}
```

### **Rule Implementations**

Each of these classes represents a specific rule used to calculate points for a receipt. These rules are automatically registered with Spring, thanks to the `@Component` annotation.s

```java
package ReceiptProcessor.application.utility;

import java.time.LocalDateTime;

import org.springframework.stereotype.Component;

import ReceiptProcessor.api.dto.AddItemInput;
import ReceiptProcessor.api.dto.AddReceiptInput;

/**
 * Rule interface defines the contract for all rules that calculate points
 * based on the provided receipt.
 */
interface Rule {
  /**
   * This method calculates points based on the provided receipt.
   *
   * @param receipt The receipt on which points are calculated.
   * @return The calculated points based on the rule logic.
   */
  int calculate(AddReceiptInput receipt);
}

// Rule 1: One point for every alphanumeric character in the retailer name.
// This rule is applied to the retailer's name and counts the alphanumeric
// characters.
// It is annotated as @Component so Spring can automatically inject it into the
// RuleRegistry.
@Component
class AddPointsForRetailName implements Rule {
  @Override
  public int calculate(AddReceiptInput receipt) {
    // Remove all non-alphanumeric characters and return the length of the resulting
    // string
    String alphaNumericalRetailName = receipt.getStore().replaceAll("[^A-Za-z0-9]", "");
    return alphaNumericalRetailName.length();
  }
}

// Rule 2: 25 points if the total is a multiple of 0.25.
// This rule checks if the receipt total is a multiple of 0.25. If so, it awards
// 50 points.
// Annotated with @Component to register it with Spring's DI container.
@Component
class AddPointsForRoundDollarTotal implements Rule {
  @Override
  public int calculate(AddReceiptInput receipt) {
    // If the total is a whole number (i.e., no decimal points), award 50 points
    return receipt.getTotal() % 1 == 0 ? 50 : 0;
  }
}

// Rule 3: 5 points for every two items on the receipt.
// This rule awards points based on the number of items on the receipt.
@Component
class AddPointsForMultipleOfQuarter implements Rule {
  @Override
  public int calculate(AddReceiptInput receipt) {
    // Award 25 points if the total is a multiple of 0.25
    return receipt.getTotal() % .25 == 0 ? 25 : 0;
  }
}

// Rule 4: If the length of the trimmed item description is a multiple of 3,
// multiply the item price by 0.2 and round up to the nearest integer to
// determine
// the points for that item.
@Component
class AddPointsForItemCount implements Rule {
  @Override
  public int calculate(AddReceiptInput receipt) {
    // Calculate points based on item count; the exact logic depends on how items
    // are structured.
    // For this example, it awards points for every two items on the receipt.
    int x = (int) Math.floor(receipt.getItems().size() / 2);
    return x * 5;
  }
}

// Rule 5: 6 points if the day in the purchase date is odd (e.g., 1st, 3rd, 5th,
// etc.).
// This rule checks the day of the month from the purchase date and awards
// points if it's odd.
@Component
class AddPointsForItemDescriptions implements Rule {
  @Override
  public int calculate(AddReceiptInput receipt) {
    int points = 0;

    // Loop through each item and apply the rule if the description length is a
    // multiple of 3
    for (AddItemInput item : receipt.getItems()) {
      if (item.getName().trim().length() % 3 == 0) {
        // Multiply the price by 2 and round the result
        points += (int) Math.round(item.getPrice() * 2);
      }
    }

    return points;
  }
}

// Rule 6: 10 points if the purchase time is between 2:00 PM and 4:00 PM
// (inclusive).
// This rule checks the purchase time and awards points if the time falls within
// the specified range.
@Component
class AddPointsForOddPurchaseDate implements Rule {
  @Override
  public int calculate(AddReceiptInput receipt) {
    // Check if the day of the month is odd (1st, 3rd, etc.)
    if (LocalDateTime.parse(receipt.getDate()).getDayOfMonth() % 2 != 0) {
      return 6; // Return 6 points if the day is odd
    }
    return 0; // No points if the day is not odd
  }
}

// Rule 7: 10 points if the purchase time is between 2:00 PM and 4:00 PM
// (inclusive).
// This rule checks if the time of the purchase is within a specific range and
// awards points.
@Component
class AddPointsForPurchaseTime implements Rule {
  @Override
  public int calculate(AddReceiptInput receipt) {
    // Check if the hour of the purchase time is between 14:00 (2:00 PM) and 16:00
    // (4:00 PM)
    int hour = LocalDateTime.parse(receipt.getDate()).getHour();
    if (hour >= 14 && hour <= 16) {
      return 10; // Award 10 points for purchases made between 2:00 PM and 4:00 PM
    }
    return 0; // No points outside of this time range
  }
}
```

---

## Additional Notes

- **Spring Dependency Injection**: The use of `@Component` and `@Autowired` allows Spring to automatically manage the injection of dependencies. This makes adding and removing rules very straightforward.
- **Extensibility**: New rules can be added simply

by creating new classes that implement the `Rule` interface and annotating them with `@Component`. The `RuleRegistry` will automatically detect them and include them in the point calculation.

- **Testability**: The rule-based design makes testing each rule in isolation easy, as each rule can be tested independently of the others.

This structure provides flexibility, maintainability, and easy extensibility while making the logic for calculating points clean and modular.
