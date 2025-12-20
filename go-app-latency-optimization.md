# Go API Latency Optimization

> **TLDR**: Used AI to identify bottlenecks and better approach, reducing critical call latency by 87.6% (from ~50s to ~6s).

> **Domain:** Backend | Infrastructure | Cloud Security | AI

> **Technologies:** Go, Google Cloud Platform (GKE), Azure Graph API, Google Directory API, Goroutines

## Context

As part of a large-scale platform migration to version 2 (Tiny Homes v2) on Google Kubernetes Engine (GKE), the Cloud Locksmith application—responsible for managing group-based permission elevations—encountered severe performance degradation. The application facilitates secure, temporary access grants across multi-cloud environments, integrating deeply with identity providers for both Azure and GCP.

## Problem/Issue/Bottleneck

After migrating to the v2 Kubernetes network mesh, the application began experiencing frequent 503 Service Unavailable errors and 504 Gateway Timeouts. Detailed investigation revealed several critical bottlenecks:

- **Inefficient Identity Lookups:** The application used `SearchMemberGroups`, which paginated through hundreds of user groups to verify a single membership, resulting in linear time complexity based on group count.
- **Redundant Revocation Logic:** The system repeatedly attempted to remove non-existent users from groups without prior existence checks, causing unnecessary API wait times.
- **Sequential Execution:** Access revocations were processed one by one, creating a massive queue delay during bulk operations.
- **Strict Network Thresholds:** The v2 network mesh enforced a 45-second timeout, which the original 50-second sequential process consistently violated.

## Solution

I used AI to identify the bottlenecks, by adding fine-grained logging and asking insightful questions to find better approaches to the problematic pieces of code. After that, I applied a refactor the core integration logic and concurrency model to move from an "iterative-filtering" approach to a "direct-targeted" approach:

1. **Targeted API Migration:** Replaced comprehensive group listing with direct membership check endpoints. For Azure, I migrated to the `checkMemberGroups` endpoint; for GCP, I implemented `Members.Get`. This reduced the payload and processing time from to per check.
2. **Defensive Membership Logic:** Introduced `UserExists` and `IsMemberOfGroup` checks to skip redundant "remove" calls for users who were already revoked or non-existent, eliminating the 48+ second delays caused by waiting for API failure responses.
3. **Concurrent Revocation Model:** Implemented a worker-pool pattern using Go's `goroutines` and `sync.WaitGroup`. This allowed the system to process multiple revocation grants in parallel rather than sequentially.
4. **Resilient Error Handling:** Standardized error mapping for 404 responses to distinguish between "User not found" (a valid business state for revocation) and actual API failures.

## Impact

- **Performance:** Reduced critical call latency from ~50 seconds to ~6.2 seconds, an **87.6% improvement**.
- **Reliability:** Successfully mitigated all 504 Gateway Timeouts by bringing execution time well under the 45-second network threshold.
- **Scalability:** Bulk revocation throughput increased significantly due to the concurrent processing model, allowing the system to handle spikes in access expiration without queuing.
- **Efficiency:** Drastically reduced the number of outbound API calls to Google and Azure, lowering resource consumption and mitigating the risk of API rate limiting.

## Diff

### Optimized Membership Checks

The following refactor replaces expensive group listing with a direct, single-call membership check for Azure AD.

```go
// locksmith/pkg/apis/azure/graph.go

// IsUserMemberOfGroup checks if a user is a member of a specific group using an optimized API call
// Reference: https://learn.microsoft.com/en-us/graph/api/directoryobject-checkmembergroups
func IsUserMemberOfGroup(member, groupID, token string) (bool, error) {
	userID, err := GetUserId(member, token)
	if err != nil {
		return false, err
	}

	url := fmt.Sprintf("https://graph.microsoft.com/v1.0/users/%s/checkMemberGroups", userID)
	requestBody := map[string]interface{}{
		"groupIds": []string{groupID},
	}

	jsonBody, _ := json.Marshal(requestBody)
	body, err := Request("POST", url, token, bytes.NewBuffer(jsonBody))
	if err != nil {
		return false, errors.ErrAzureCheckMemberGroups
	}

	var response struct {
		Value []string `json:"value"`
	}
	json.Unmarshal(body, &response)

	for _, id := range response.Value {
		if id == groupID {
			return true, nil
		}
	}
	return false, nil
}

```

### Concurrent Revocation Implementation

The `RunRevokeOperation` was refactored to utilize goroutines, allowing independent grants to be revoked in parallel.

```go
// locksmith/pkg/application/app.go

func (a App) RunRevokeOperation() (models.ElevationRequestList, error) {
	grants, err := a.ChangeRetriever.GetActiveGrants()
	// ... error handling ...

	revokedChan := make(chan *models.ElevationRequest, len(grants))
	var wg sync.WaitGroup

	for _, grant := range grants {
		wg.Add(1)
		go func(g *models.ElevationRequest) {
			defer wg.Done()

			// Business logic to determine if grant should be revoked
			r, err := a.ChangeRetriever.Query(g.ChangeIncident)
			if err != nil { return }

			if snowfield.ShouldRevoke(g, r.(snowfield.Record)) {
				if err := a.RevokeElevationRequest(g); err == nil {
					revokedChan <- g
				}
			}
		}(grant)
	}

	go func() {
		wg.Wait()
		close(revokedChan)
	}()

	for revokedGrant := range revokedChan {
		revoked = append(revoked, revokedGrant)
	}
	return revoked, nil
}

```

### Direct Google Directory API Access

Instead of fetching all groups a user belongs to, we now target the specific group resource directly.

```go
// locksmith/pkg/apis/directorylookup/memberdirectory/memberdirectory.go

func (md *MemberDirectory) IsMemberOfGroup(groupKey, memberKey string) (bool, error) {
	// Single API call instead of paginating through all user's groups
	_, err := md.adminService.Members.Get(groupKey, memberKey).Do()

	if err != nil {
		if containsAny(err.Error(), []string{"not found", "404"}) {
			return false, nil
		}
		return false, errors.ErrAdminServiceGroupsListFailed
	}
	return true, nil
}

```
