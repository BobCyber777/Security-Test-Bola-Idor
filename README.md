# Security-Test-Bola-Idor
Broken Object level Authorization Confirmed

## Security Test: IDOR / BOLA — `/api/user/<user_id>`

**Status:** 🔴 Vulnerability Confirmed
**Environment:** Authorized local security-testing lab
**Endpoint:** `GET /api/user/<user_id>`

### Test Summary

The `/api/user/<user_id>` endpoint was tested with invalid, valid, and alternate user IDs.

Invalid IDs consistently returned:

```text
HTTP/1.1 404 NOT FOUND
{"error":"user not found"}
```

A valid user ID returned:

```text
GET /api/user/1001
HTTP/1.1 200 OK
{"email":"alice@lab.local","id":"1001","name":"Alice"}
```

Changing the object ID to another valid user:

```text
GET /api/user/1002
HTTP/1.1 200 OK
{"email":"bob@lab.local","id":"1002","name":"Bob"}
```

No authentication or object-ownership authorization check exists in the route.

### Finding

**Broken Object-Level Authorization (BOLA), commonly referred to as IDOR, is confirmed.**

An unauthenticated caller can retrieve another user's record by changing the `user_id` in the URL.

### Root Cause

The route directly retrieves the requested object:

```python
user = users.get(user_id)
```

and returns the record without verifying that the requester is authenticated and authorized to access that specific user.

### Recommended Fix

Implement authentication and object-level authorization before returning the requested resource.

Conceptually:

```python
@app.route("/api/user/<user_id>")
@login_required
def get_user(user_id):
    user = users.get(user_id)

    if not user:
        return jsonify({"error": "user not found"}), 404

    if user_id != current_user.id:
        return jsonify({"error": "forbidden"}), 403

    return jsonify({
        "id": user_id,
        "name": user["name"],
        "email": user["email"]
    })
```

The exact authorization mechanism should match the application's authentication architecture.

### Verification After Fix

Retest:

```text
Authenticated user → own resource     → 200 OK
Authenticated user → another user's resource → 403 Forbidden
Unauthenticated → protected resource  → 401 Unauthorized
Nonexistent ID → 404 Not Found
```

The objective is to ensure that knowing or changing a resource ID is **never sufficient to obtain another user's data**.

### Conclusion

The lab successfully demonstrated a reproducible **IDOR/BOLA vulnerability** caused by missing authentication and object-level authorization. Invalid-input handling was functioning correctly, but access control at the resource level was absent.
