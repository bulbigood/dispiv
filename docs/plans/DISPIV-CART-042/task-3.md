---
task_id: t3_api
pipeline_id: DISPIV-CART-042
emc_phase: migrate
depends_on: [t1_dto, t2_db]
breaks_compilation: true
---

# Task 3: API Refactor (Migrate)

## Target Files

| File | Action | Status |
|:---|:---|:---|
| `src/.../controller/CartController.java` | Modify | Existing |
| `src/.../service/CartService.java` | Modify | Existing |
| `src/.../service/LegacyCartService.java` | Delete | Phase contract |

## Context & Goal

Выполни фазу **Migrate**. Переведи контроллеры и сервисы на новые DTO и репозитории, 
созданные в T1 и T2. Это ломающее изменение — старый API временно не работает.

## Preconditions (from depends_on)

- [x] T1: `CartItemRequest`, `CartState` существуют в пакете `dto`
- [x] T2: `CartItemRepository`, `CartItemEntity` существуют в пакете `db`

## Interface & Signatures

### File: `CartController.java` (Modify)

```java
// REMOVE:
@PostMapping("/api/v1/cart/add")
public ResponseEntity<?> addToCart(@RequestBody AddToCartForm form) { ... }

// ADD:
@PostMapping("/api/v3/cart/items")
public ResponseEntity<CartItemResponse> addItem(@RequestBody CartItemRequest request) {
    CartItemEntity entity = cartService.addItem(request);
    return ResponseEntity.ok(CartItemResponse.fromEntity(entity));
}
```

## Logical Specification

### State Machine Transitions

| From | To | Trigger | Guard |
|:---|:---|:---|:---|
| ACTIVE | PAID | `checkout()` | `items.size() > 0` |
| ACTIVE | LOCKED | `lock()` | — |
| LOCKED | ACTIVE | `unlock()` | — |

### Error Handling

| Condition | HTTP Status | Error Code |
|:---|:---|:---|
| Cart is PAID | 409 Conflict | `CART_ALREADY_PAID` |
| Item not found | 404 Not Found | `ITEM_NOT_FOUND` |
| Cart is LOCKED | 423 Locked | `CART_LOCKED` |

## Compilation Impact

⚠️ После этой задачи следующие файлы **не компилируются** до T4:
- `LegacyCartService.java` — использует удалённые типы (будет удалён в T4)
- `OldCartControllerTest.java` — ссылается на `/api/v1/cart/add`

## Rollback Strategy

Если задача провалилась:
1. `git checkout HEAD -- src/.../controller/CartController.java`
2. `git checkout HEAD -- src/.../service/CartService.java`
3. Вернуться к состоянию после T1+T2