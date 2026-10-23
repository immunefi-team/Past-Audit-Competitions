# #49250 \[SC-Insight] \`AccessControl\`: unnecessary box usage in \`\_grant\_role\`

**Submitted on Jul 13th 2025 at 19:10:11 UTC by @ustas for** [**Audit Comp | Folks Smart Contract Library**](https://immunefi.com/audit-competition/folks-sc-library)

* **Report ID:** #49250
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Folks-Finance/algorand-smart-contract-library/blob/main/contracts/library/AccessControl.py>
* **Impacts:**

## Description

## Description

When `grant_role` is called, the internal `_grant_role` subroutine explicitly sets the admin of a `role` to `default_admin_role`.

```py
    @subroutine
    def _grant_role(self, role: Bytes16, account: Address) -> Bool:
        # if new role then add the default admin role
        if role not in self.roles:
            self.roles[role] = self.default_admin_role()
```

However, this explicit storage change is redundant. The `get_role_admin` function returns `default_admin_role` if a role's admin is not found in the `roles` box. This provides an implicit default.

```py
    @abimethod(readonly=True)
    def get_role_admin(self, role: Bytes16) -> Bytes16:
        """Returns the admin role that controls a role

        Args:
            role: The role to get its admin of

        Returns:
            The role admin
        """
        if role not in self.roles:
            return self.default_admin_role()
        return self.roles[role]
```

## Remediation

Remove the `if` condition block in `_grant_role`.

## Proof of Concept

## Proof of Concept

1. Add any new role
2. There's a storage write
