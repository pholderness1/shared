# Ping AIC User Reconciliation and Validation Plan

## Overview

Ping AIC will serve as the identity store, with a CSV file used to provide the user data. The CSV file will contain a combination of existing users and new users.

To reduce risk and ensure the data is processed correctly, reconciliation will be performed in two separate passes. Each pass will be followed by validation to confirm that the expected users and their attributes have been processed correctly, including any resulting updates to the downstream Active Directory (AD) system.

## Process Steps

### 1\. Prepare the CSV File

The CSV file will contain both existing users and new users. The file will be divided into two groups:

- **Pass 1:** Existing users only.
- **Pass 2:** All remaining users, primarily new users.

This separation allows existing user records to be reconciled and validated before processing the remaining records.

### 2\. Reconcile Existing Users

The first reconciliation pass will process only the existing users in Ping AIC.

Once the reconciliation is complete, validation will be performed to confirm that the results match expectations.

**Validation activities:**

- Confirm that the number of users processed matches the expected count.
- Spot-check user attribute values to ensure that the data was populated or updated correctly.
- Validate the implicit synchronization operations sent to the downstream AD system, confirming that the expected changes occurred.

**The second reconciliation pass will not begin until these validation checks are complete and the results are acceptable.**

### 3\. Reconcile the Remaining Users

After the first pass has been validated, the second reconciliation pass will process the remaining users from the CSV file.

The same validation approach will be applied to this pass.

**Validation activities:**

- Confirm that the number of users processed matches the expected count for the second group.
- Spot-check user attribute values to verify data accuracy.
- Validate the resulting implicit synchronization operations and confirm that the expected changes are reflected in the downstream AD system.

### 4\. Confirm Overall Results

After both reconciliation passes and their respective validation activities are complete, confirm that the overall results align with expectations.

This includes verifying that:

- Both groups of users were processed successfully.
- User counts match the expected totals.
- Sampled user attributes contain the correct values.
- Downstream AD updates resulting from implicit synchronization have been validated.
