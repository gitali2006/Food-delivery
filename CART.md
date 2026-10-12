# QuickBite — MEMBER 4: Cart & Checkout — COMPLETE CODE

*Backend (Java / Spring Boot) + Frontend (HTML + CSS) of this module in one document.*

---

## PART A — BACKEND (Java)

### `src/main/java/com/example/fooddelivery/model/CartItem.java`
```java
package com.example.fooddelivery.model;

import com.example.fooddelivery.common.BaseEntity;
import jakarta.persistence.Entity;
import jakarta.persistence.JoinColumn;
import jakarta.persistence.ManyToOne;
import jakarta.persistence.Table;

import java.math.BigDecimal;

/**
 * MEMBER 4 - CART MODULE
 * One row = "this customer wants N portions of this menu item".
 */
@Entity
@Table(name = "cart_items")
public class CartItem extends BaseEntity {

    @ManyToOne
    @JoinColumn(name = "customer_id", nullable = false)
    private Customer customer;

    @ManyToOne
    @JoinColumn(name = "menu_item_id", nullable = false)
    private MenuItem menuItem;

    private int quantity = 1;

    public CartItem() {
    }

    public CartItem(Customer customer, MenuItem menuItem, int quantity) {
        this.customer = customer;
        this.menuItem = menuItem;
        this.quantity = quantity;
    }

    @Override
    public String getDisplayName() {
        return menuItem != null ? menuItem.getName() + " x" + quantity : "Cart item";
    }

    /** Line total = unit price x quantity. */
    public BigDecimal getLineTotal() {
        if (menuItem == null) {
            return BigDecimal.ZERO;
        }
        return menuItem.getPrice().multiply(BigDecimal.valueOf(quantity));
    }

    public void increaseQuantity(int amount) {
        this.quantity += amount;
    }

    // ---------- Getters / Setters ----------

    public Customer getCustomer() {
        return customer;
    }

    public void setCustomer(Customer customer) {
        this.customer = customer;
    }

    public MenuItem getMenuItem() {
        return menuItem;
    }

    public void setMenuItem(MenuItem menuItem) {
        this.menuItem = menuItem;
    }

    public int getQuantity() {
        return quantity;
    }

    public void setQuantity(int quantity) {
        this.quantity = quantity;
    }
}

```

### `src/main/java/com/example/fooddelivery/repository/CartItemRepository.java`
```java
package com.example.fooddelivery.repository;

import com.example.fooddelivery.model.CartItem;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;

/** MEMBER 4 - data access for cart items. */
@Repository
public interface CartItemRepository extends JpaRepository<CartItem, Long> {

    List<CartItem> findByCustomerId(Long customerId);

    Optional<CartItem> findByCustomerIdAndMenuItemId(Long customerId, Long menuItemId);

    void deleteByCustomerId(Long customerId);

    long countByCustomerId(Long customerId);
}

```

### `src/main/java/com/example/fooddelivery/service/CartService.java`
```java
package com.example.fooddelivery.service;

import com.example.fooddelivery.model.CartItem;
import com.example.fooddelivery.model.MenuItem;
import com.example.fooddelivery.model.Restaurant;

import java.math.BigDecimal;
import java.util.List;
import java.util.Optional;

public interface CartService {

    List<CartItem> getCartItems(Long customerId);

    CartItem addToCart(Long customerId, Long menuItemId, int quantity);

    CartItem updateQuantity(Long cartItemId, int quantity);

    void removeItem(Long cartItemId);

    void clearCart(Long customerId);

    BigDecimal getCartTotal(Long customerId);

    int getCartItemCount(Long customerId);

    Optional<Restaurant> getCartRestaurant(Long customerId);

    /** Delivery fee rule: free above Rs. 3000, otherwise Rs. 300. */
    BigDecimal getDeliveryFee(Long customerId);

    BigDecimal getGrandTotal(Long customerId);

    List<MenuItem> suggestionsForCart(Long customerId);
}

```

### `src/main/java/com/example/fooddelivery/service/impl/CartServiceImpl.java`
```java
package com.example.fooddelivery.service.impl;

import com.example.fooddelivery.common.ResourceNotFoundException;
import com.example.fooddelivery.model.CartItem;
import com.example.fooddelivery.model.Customer;
import com.example.fooddelivery.model.MenuItem;
import com.example.fooddelivery.model.Restaurant;
import com.example.fooddelivery.repository.CartItemRepository;
import com.example.fooddelivery.repository.CustomerRepository;
import com.example.fooddelivery.repository.MenuItemRepository;
import com.example.fooddelivery.service.CartService;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.List;
import java.util.Optional;

/**
 * MEMBER 4 - business logic for the Cart module.
 * Rules enforced here:
 *  - one cart can only contain items from ONE restaurant (like real delivery apps)
 *  - a sold-out item cannot be added
 *  - adding the same dish twice increases the quantity instead of duplicating the row
 */
@Service
public class CartServiceImpl implements CartService {

    private static final BigDecimal DELIVERY_FEE = new BigDecimal("300");
    private static final BigDecimal FREE_DELIVERY_ABOVE = new BigDecimal("3000");

    private final CartItemRepository cartItemRepository;
    private final CustomerRepository customerRepository;
    private final MenuItemRepository menuItemRepository;

    public CartServiceImpl(CartItemRepository cartItemRepository,
                           CustomerRepository customerRepository,
                           MenuItemRepository menuItemRepository) {
        this.cartItemRepository = cartItemRepository;
        this.customerRepository = customerRepository;
        this.menuItemRepository = menuItemRepository;
    }

    @Override
    public List<CartItem> getCartItems(Long customerId) {
        return cartItemRepository.findByCustomerId(customerId);
    }

    @Override
    @Transactional
    public CartItem addToCart(Long customerId, Long menuItemId, int quantity) {
        Customer customer = customerRepository.findById(customerId)
                .orElseThrow(() -> ResourceNotFoundException.of("Customer", customerId));
        MenuItem menuItem = menuItemRepository.findById(menuItemId)
                .orElseThrow(() -> ResourceNotFoundException.of("Menu item", menuItemId));

        if (!menuItem.isAvailable()) {
            throw new IllegalStateException(menuItem.getName() + " is sold out right now.");
        }
        if (quantity < 1) {
            quantity = 1;
        }

        // Rule: only one restaurant per cart.
        Optional<Restaurant> current = getCartRestaurant(customerId);
        if (current.isPresent() && !current.get().getId().equals(menuItem.getRestaurant().getId())) {
            cartItemRepository.deleteByCustomerId(customerId);
        }

        Optional<CartItem> existing =
                cartItemRepository.findByCustomerIdAndMenuItemId(customerId, menuItemId);
        if (existing.isPresent()) {
            CartItem cartItem = existing.get();
            cartItem.increaseQuantity(quantity);
            return cartItemRepository.save(cartItem);
        }

        CartItem cartItem = new CartItem(customer, menuItem, quantity);
        return cartItemRepository.save(cartItem);
    }

    @Override
    public CartItem updateQuantity(Long cartItemId, int quantity) {
        CartItem cartItem = cartItemRepository.findById(cartItemId)
                .orElseThrow(() -> ResourceNotFoundException.of("Cart item", cartItemId));
        if (quantity < 1) {
            cartItemRepository.delete(cartItem);
            return null;
        }
        cartItem.setQuantity(quantity);
        return cartItemRepository.save(cartItem);
    }

    @Override
    public void removeItem(Long cartItemId) {
        if (!cartItemRepository.existsById(cartItemId)) {
            throw ResourceNotFoundException.of("Cart item", cartItemId);
        }
        cartItemRepository.deleteById(cartItemId);
    }

    @Override
    @Transactional
    public void clearCart(Long customerId) {
        cartItemRepository.deleteByCustomerId(customerId);
    }

    @Override
    public BigDecimal getCartTotal(Long customerId) {
        BigDecimal total = BigDecimal.ZERO;
        for (CartItem item : getCartItems(customerId)) {
            total = total.add(item.getLineTotal());
        }
        return total;
    }

    @Override
    public int getCartItemCount(Long customerId) {
        int count = 0;
        for (CartItem item : getCartItems(customerId)) {
            count += item.getQuantity();
        }
        return count;
    }

    @Override
    public Optional<Restaurant> getCartRestaurant(Long customerId) {
        List<CartItem> items = getCartItems(customerId);
        if (items.isEmpty()) {
            return Optional.empty();
        }
        return Optional.of(items.get(0).getMenuItem().getRestaurant());
    }

    @Override
    public BigDecimal getDeliveryFee(Long customerId) {
        BigDecimal total = getCartTotal(customerId);
        if (total.signum() == 0 || total.compareTo(FREE_DELIVERY_ABOVE) >= 0) {
            return BigDecimal.ZERO;
        }
        return DELIVERY_FEE;
    }

    @Override
    public BigDecimal getGrandTotal(Long customerId) {
        return getCartTotal(customerId).add(getDeliveryFee(customerId));
    }

    /** "You may also like" - cheap dishes from the same restaurant, not yet in the cart. */
    @Override
    public List<MenuItem> suggestionsForCart(Long customerId) {
        Optional<Restaurant> restaurant = getCartRestaurant(customerId);
        if (restaurant.isEmpty()) {
            return List.of();
        }
        List<Long> inCart = new ArrayList<>();
        for (CartItem item : getCartItems(customerId)) {
            inCart.add(item.getMenuItem().getId());
        }
        List<MenuItem> suggestions = new ArrayList<>();
        for (MenuItem item : menuItemRepository.findByRestaurantIdAndAvailableTrue(restaurant.get().getId())) {
            if (!inCart.contains(item.getId()) && suggestions.size() < 4) {
                suggestions.add(item);
            }
        }
        return suggestions;
    }
}

```

### `src/main/java/com/example/fooddelivery/controller/CartController.java`
```java
package com.example.fooddelivery.controller;

import com.example.fooddelivery.common.SessionHelper;
import com.example.fooddelivery.model.CartItem;
import com.example.fooddelivery.model.Order;
import com.example.fooddelivery.model.enums.PaymentMethod;
import com.example.fooddelivery.service.CartService;
import com.example.fooddelivery.service.OrderService;
import com.example.fooddelivery.service.PaymentService;
import jakarta.servlet.http.HttpSession;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;

import java.util.List;

/**
 * MEMBER 4 - CART MODULE
 * READ   -> GET  /cart                (view the cart)
 * CREATE -> POST /cart/add            (add a dish)
 * UPDATE -> POST /cart/update         (change quantity)
 * DELETE -> POST /cart/remove/{id}    (remove one line) + POST /cart/clear (empty cart)
 * + GET/POST /cart/checkout -> converts the cart into an order (hand-over to Member 5).
 */
@Controller
@RequestMapping("/cart")
public class CartController {

    private final CartService cartService;
    private final OrderService orderService;
    private final PaymentService paymentService;

    public CartController(CartService cartService,
                          OrderService orderService,
                          PaymentService paymentService) {
        this.cartService = cartService;
        this.orderService = orderService;
        this.paymentService = paymentService;
    }

    @GetMapping
    public String viewCart(HttpSession session, Model model) {
        Long customerId = SessionHelper.getCustomerId(session);
        // always defined, so the template can safely use it in negative conditions
        model.addAttribute("needLogin", customerId == null);
        if (customerId == null) {
            return "cart/cart";
        }
        List<CartItem> items = cartService.getCartItems(customerId);
        model.addAttribute("cartItems", items);
        model.addAttribute("subtotal", cartService.getCartTotal(customerId));
        model.addAttribute("deliveryFee", cartService.getDeliveryFee(customerId));
        model.addAttribute("grandTotal", cartService.getGrandTotal(customerId));
        model.addAttribute("restaurant", cartService.getCartRestaurant(customerId).orElse(null));
        model.addAttribute("suggestions", cartService.suggestionsForCart(customerId));
        return "cart/cart";
    }

    @PostMapping("/add")
    public String addToCart(@RequestParam Long menuItemId,
                            @RequestParam(defaultValue = "1") int quantity,
                            HttpSession session,
                            RedirectAttributes redirectAttributes) {
        Long customerId = SessionHelper.getCustomerId(session);
        if (customerId == null) {
            redirectAttributes.addFlashAttribute("error", "Please sign in first to start ordering.");
            return "redirect:/login";
        }
        try {
            CartItem item = cartService.addToCart(customerId, menuItemId, quantity);
            redirectAttributes.addFlashAttribute("success",
                    item.getMenuItem().getName() + " added to your cart.");
        } catch (IllegalStateException ex) {
            redirectAttributes.addFlashAttribute("error", ex.getMessage());
        }
        return "redirect:/menu";
    }

    @PostMapping("/update")
    public String updateQuantity(@RequestParam Long cartItemId,
                                 @RequestParam int quantity,
                                 RedirectAttributes redirectAttributes) {
        cartService.updateQuantity(cartItemId, quantity);
        redirectAttributes.addFlashAttribute("success", "Cart updated.");
        return "redirect:/cart";
    }

    @PostMapping("/remove/{id}")
    public String remove(@PathVariable Long id, RedirectAttributes redirectAttributes) {
        cartService.removeItem(id);
        redirectAttributes.addFlashAttribute("success", "Item removed from cart.");
        return "redirect:/cart";
    }

    @PostMapping("/clear")
    public String clear(HttpSession session, RedirectAttributes redirectAttributes) {
        Long customerId = SessionHelper.getCustomerId(session);
        if (customerId != null) {
            cartService.clearCart(customerId);
        }
        redirectAttributes.addFlashAttribute("success", "Cart cleared.");
        return "redirect:/cart";
    }

    // ---------------- CHECKOUT ----------------
    @GetMapping("/checkout")
    public String checkoutPage(HttpSession session, Model model, RedirectAttributes redirectAttributes) {
        Long customerId = SessionHelper.getCustomerId(session);
        if (customerId == null) {
            redirectAttributes.addFlashAttribute("error", "Please sign in first.");
            return "redirect:/login";
        }
        List<CartItem> items = cartService.getCartItems(customerId);
        if (items.isEmpty()) {
            redirectAttributes.addFlashAttribute("error", "Your cart is empty.");
            return "redirect:/cart";
        }
        model.addAttribute("cartItems", items);
        model.addAttribute("subtotal", cartService.getCartTotal(customerId));
        model.addAttribute("deliveryFee", cartService.getDeliveryFee(customerId));
        model.addAttribute("grandTotal", cartService.getGrandTotal(customerId));
        model.addAttribute("paymentMethods", PaymentMethod.values());
        return "cart/checkout";
    }

    /** Creates the order from the cart and the matching payment record. */
    @PostMapping("/checkout")
    public String placeOrder(@RequestParam String deliveryAddress,
                             @RequestParam(required = false) String notes,
                             @RequestParam PaymentMethod paymentMethod,
                             HttpSession session,
                             RedirectAttributes redirectAttributes) {
        Long customerId = SessionHelper.getCustomerId(session);
        if (customerId == null) {
            redirectAttributes.addFlashAttribute("error", "Please sign in first.");
            return "redirect:/login";
        }
        if (deliveryAddress == null || deliveryAddress.isBlank()) {
            redirectAttributes.addFlashAttribute("error", "Delivery address is required.");
            return "redirect:/cart/checkout";
        }
        try {
            Order order = orderService.placeOrderFromCart(customerId, deliveryAddress, notes);
            paymentService.createForOrder(order, paymentMethod);
            redirectAttributes.addFlashAttribute("success",
                    "Order " + order.getOrderNumber() + " placed successfully!");
            return "redirect:/orders/" + order.getId();
        } catch (IllegalStateException ex) {
            redirectAttributes.addFlashAttribute("error", ex.getMessage());
            return "redirect:/cart";
        }
    }
}

```


## PART B — FRONTEND (HTML templates)

### `src/main/resources/templates/cart/cart.html`
```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>QuickBite - My cart (Member 4)</title>
    <link rel="stylesheet" th:href="@{/css/style.css}">
</head>
<body>
<div th:replace="~{fragments/layout :: nav}"></div>

<main class="container">
    <div class="page-head">
        <div>
            <h1>My cart</h1>
            <p class="muted">Member 4 &mdash; cart CRUD: add items, change quantities, remove items, clear cart, checkout.</p>
        </div>
        <a th:href="@{/menu}" class="btn btn-outline">+ Add more food</a>
    </div>

    <!-- not signed in -->
    <div class="card empty" th:if="${needLogin}">
        <span class="big">&#128274;</span>
        Please sign in to use the cart.
        <div class="btn-row" style="justify-content:center;margin-top:14px;">
            <a th:href="@{/login}" class="btn">Sign in</a>
            <a th:href="@{/customers}" class="btn btn-outline">Sign in as a demo customer</a>
        </div>
    </div>

    <!-- empty cart -->
    <div class="card empty" th:if="${!needLogin and #lists.isEmpty(cartItems)}">
        <span class="big">&#128722;</span>
        Your cart is empty.
        <div class="btn-row" style="justify-content:center;margin-top:14px;">
            <a th:href="@{/menu}" class="btn btn-orange">Browse the menu</a>
        </div>
    </div>

    <!-- cart with items -->
    <div th:if="${!needLogin and !#lists.isEmpty(cartItems)}">
        <div class="card tight mb">
            <strong>Ordering from:</strong>
            <span th:text="${restaurant != null ? restaurant.name : '-'}">Restaurant</span>
            <span class="muted">&mdash; one cart can hold dishes from one restaurant only.</span>
        </div>

        <div class="table-wrap">
            <table class="table">
                <thead>
                <tr>
                    <th>Dish</th>
                    <th>Unit price (Rs.)</th>
                    <th>Quantity</th>
                    <th>Line total (Rs.)</th>
                    <th></th>
                </tr>
                </thead>
                <tbody>
                <tr th:each="item : ${cartItems}">
                    <td>
                        <strong th:text="${item.menuItem.name}">Chicken Kottu</strong>
                        <div class="muted" style="font-size:.82rem;" th:text="${item.menuItem.restaurant.name}">Restaurant</div>
                    </td>
                    <td th:text="${#numbers.formatDecimal(item.menuItem.price, 1, 'COMMA', 2, 'POINT')}">0.00</td>
                    <td>
                        <form th:action="@{/cart/update}" method="post" style="display:flex;gap:6px;align-items:center;">
                            <input type="hidden" name="cartItemId" th:value="${item.id}">
                            <input class="qty-input" type="number" name="quantity" th:value="${item.quantity}" min="1">
                            <button type="submit" class="btn btn-sm btn-slate">Update</button>
                        </form>
                    </td>
                    <td th:text="${#numbers.formatDecimal(item.lineTotal, 1, 'COMMA', 2, 'POINT')}">0.00</td>
                    <td>
                        <form th:action="@{/cart/remove/{id}(id=${item.id})}" method="post" class="inline-form">
                            <button type="submit" class="btn btn-sm btn-danger">Remove</button>
                        </form>
                    </td>
                </tr>
                </tbody>
            </table>
        </div>

        <div class="card totals mt">
            <div class="row"><span>Subtotal</span>
                <span th:text="'Rs. ' + ${#numbers.formatDecimal(subtotal, 1, 'COMMA', 2, 'POINT')}">Rs. 0.00</span></div>
            <div class="row"><span>Delivery fee</span>
                <span th:text="'Rs. ' + ${#numbers.formatDecimal(deliveryFee, 1, 'COMMA', 2, 'POINT')}">Rs. 0.00</span></div>
            <div class="row grand"><span>Total</span>
                <span th:text="'Rs. ' + ${#numbers.formatDecimal(grandTotal, 1, 'COMMA', 2, 'POINT')}">Rs. 0.00</span></div>
            <p class="muted" style="font-size:.82rem;">Delivery is free for orders above Rs. 3,000.</p>
            <div class="btn-row">
                <a th:href="@{/cart/checkout}" class="btn btn-orange">Proceed to checkout</a>
                <form th:action="@{/cart/clear}" method="post" class="inline-form"
                      >
                    <button type="submit" class="btn btn-outline">Clear cart</button>
                </form>
            </div>
        </div>

        <div th:if="${!#lists.isEmpty(suggestions)}">
            <div class="section-title"><h2>You may also like</h2></div>
            <div class="card-grid">
                <div class="card dish-card" th:each="item : ${suggestions}">
                    <div class="dish-head">
                        <h3 th:text="${item.name}">Dish</h3>
                        <span class="price" th:text="'Rs. ' + ${#numbers.formatDecimal(item.price, 1, 'COMMA', 2, 'POINT')}">Rs. 0</span>
                    </div>
                    <p th:text="${item.description} ?: ''"></p>
                    <form class="dish-actions" th:action="@{/cart/add}" method="post">
                        <input type="hidden" name="menuItemId" th:value="${item.id}">
                        <input class="qty-input" type="number" name="quantity" value="1" min="1">
                        <button type="submit" class="btn btn-sm btn-orange">Add to cart</button>
                    </form>
                </div>
            </div>
        </div>
    </div>
</main>

<div th:replace="~{fragments/layout :: footer}"></div>
</body>
</html>

```

### `src/main/resources/templates/cart/checkout.html`
```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>QuickBite - Checkout</title>
    <link rel="stylesheet" th:href="@{/css/style.css}">
</head>
<body>
<div th:replace="~{fragments/layout :: nav}"></div>

<main class="container">
    <div class="page-head">
        <div>
            <h1>Checkout</h1>
            <p class="muted">Member 4 &rarr; Member 5: the cart becomes a real order, the payment record is created automatically.</p>
        </div>
        <a th:href="@{/cart}" class="btn btn-outline">&larr; Back to cart</a>
    </div>

    <div class="table-wrap">
        <table class="table">
            <thead>
            <tr>
                <th>Dish</th>
                <th>Qty</th>
                <th>Line total (Rs.)</th>
            </tr>
            </thead>
            <tbody>
            <tr th:each="item : ${cartItems}">
                <td th:text="${item.menuItem.name}">Dish</td>
                <td th:text="${item.quantity}">1</td>
                <td th:text="${#numbers.formatDecimal(item.lineTotal, 1, 'COMMA', 2, 'POINT')}">0.00</td>
            </tr>
            </tbody>
        </table>
    </div>

    <div class="card totals mt">
        <div class="row"><span>Subtotal</span>
            <span th:text="'Rs. ' + ${#numbers.formatDecimal(subtotal, 1, 'COMMA', 2, 'POINT')}">Rs. 0.00</span></div>
        <div class="row"><span>Delivery fee</span>
            <span th:text="'Rs. ' + ${#numbers.formatDecimal(deliveryFee, 1, 'COMMA', 2, 'POINT')}">Rs. 0.00</span></div>
        <div class="row grand"><span>Total to pay</span>
            <span th:text="'Rs. ' + ${#numbers.formatDecimal(grandTotal, 1, 'COMMA', 2, 'POINT')}">Rs. 0.00</span></div>
    </div>

    <div class="card mt">
        <h2>Delivery details</h2>
        <form th:action="@{/cart/checkout}" method="post">
            <div class="form-grid">
                <div class="form-group full">
                    <label for="deliveryAddress">Delivery address *</label>
                    <textarea id="deliveryAddress" name="deliveryAddress" required
                              placeholder="House number, street, city"></textarea>
                </div>
                <div class="form-group full">
                    <label for="notes">Order notes (optional)</label>
                    <input type="text" id="notes" name="notes" placeholder="e.g. no onions, ring the bell twice">
                </div>
                <div class="form-group">
                    <label for="paymentMethod">Payment method</label>
                    <select id="paymentMethod" name="paymentMethod">
                        <option th:each="method : ${paymentMethods}" th:value="${method}" th:text="${method.label}">Method</option>
                    </select>
                </div>
            </div>
            <div class="form-actions">
                <button type="submit" class="btn btn-orange">Place order</button>
                <a th:href="@{/cart}" class="btn btn-outline">Cancel</a>
            </div>
        </form>
    </div>
</main>

<div th:replace="~{fragments/layout :: footer}"></div>
</body>
</html>

```


## PART C — CSS (shared stylesheet)

All pages share `src/main/resources/static/css/style.css`. For your pages the most relevant classes are `.totals .row`, `.totals .grand`, `.qty-input`, `.dish-actions`, `.empty`. Full file below:

```css
/* ==========================================================================
   QUICKBITE - Online Food Delivery System
   Single stylesheet used by every page (pure CSS, no frameworks, no JS).
   ========================================================================== */

:root {
    --green: #16a34a;
    --green-dark: #15803d;
    --green-light: #dcfce7;
    --orange: #f97316;
    --orange-light: #ffedd5;
    --red: #dc2626;
    --red-light: #fee2e2;
    --amber: #d97706;
    --amber-light: #fef3c7;
    --blue: #2563eb;
    --blue-light: #dbeafe;
    --slate: #0f172a;
    --slate-soft: #1e293b;
    --muted: #64748b;
    --border: #e2e8f0;
    --bg: #f1f5f9;
    --card: #ffffff;
    --text: #1e293b;
    --radius: 14px;
    --shadow: 0 1px 3px rgba(15, 23, 42, .08), 0 4px 14px rgba(15, 23, 42, .06);
}

* { box-sizing: border-box; }

body {
    margin: 0;
    font-family: "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif;
    background: var(--bg);
    color: var(--text);
    line-height: 1.55;
}

.container { max-width: 1150px; margin: 0 auto; padding: 0 20px; }
main.container { padding-top: 26px; padding-bottom: 40px; min-height: 60vh; }

h1, h2, h3, h4 { color: var(--slate); line-height: 1.25; }
h1 { font-size: 1.75rem; margin: 0 0 6px; }
h2 { font-size: 1.3rem; margin: 0 0 14px; }
h3 { font-size: 1.05rem; margin: 0 0 8px; }
a { color: var(--green-dark); text-decoration: none; }
a:hover { text-decoration: underline; }
.muted { color: var(--muted); font-size: .9rem; }
.center { text-align: center; }
.mt { margin-top: 18px; }
.mb { margin-bottom: 18px; }

/* ------------------------------- navbar --------------------------------- */
.navbar {
    background: linear-gradient(90deg, var(--slate) 0%, var(--slate-soft) 100%);
    color: #fff;
    box-shadow: var(--shadow);
}
.nav-inner {
    display: flex;
    align-items: center;
    gap: 18px;
    flex-wrap: wrap;
    padding: 12px 20px;
}
.brand {
    font-size: 1.35rem;
    font-weight: 800;
    color: #fff;
    letter-spacing: .3px;
}
.brand:hover { text-decoration: none; }
.brand span { color: var(--orange); }
.nav-links { display: flex; gap: 4px; flex-wrap: wrap; flex: 1; }
.nav-links a {
    color: #e2e8f0;
    padding: 6px 11px;
    border-radius: 8px;
    font-size: .92rem;
}
.nav-links a:hover { background: rgba(255, 255, 255, .12); text-decoration: none; color: #fff; }
.nav-actions { display: flex; align-items: center; gap: 10px; }
.user-chip { display: flex; align-items: center; gap: 8px; color: #e2e8f0; font-size: .9rem; }
.user-chip form { margin: 0; }
.cart-link { color: #fff; border-color: #475569; }
.cart-link:hover { background: rgba(255,255,255,.1); }
.badge-count {
    background: var(--orange);
    color: #fff;
    border-radius: 999px;
    padding: 1px 8px;
    font-size: .78rem;
    margin-left: 4px;
}

/* ------------------------------- alerts --------------------------------- */
.alert {
    padding: 11px 16px;
    border-radius: 10px;
    margin: 12px 0 0;
    font-size: .93rem;
    border: 1px solid transparent;
}
.alert-success { background: var(--green-light); color: #14532d; border-color: #86efac; }
.alert-error { background: var(--red-light); color: #7f1d1d; border-color: #fca5a5; }
.alert-info { background: var(--blue-light); color: #1e3a8a; border-color: #93c5fd; }

/* ------------------------------- buttons -------------------------------- */
.btn {
    display: inline-block;
    background: var(--green);
    color: #fff !important;
    border: 1px solid var(--green);
    padding: 9px 16px;
    border-radius: 10px;
    font-size: .92rem;
    font-weight: 600;
    cursor: pointer;
    text-decoration: none !important;
    transition: background .15s ease, transform .05s ease;
}
.btn:hover { background: var(--green-dark); }
.btn:active { transform: translateY(1px); }
.btn-sm { padding: 5px 11px; font-size: .82rem; border-radius: 8px; }
.btn-outline { background: transparent; color: var(--green-dark) !important; }
.btn-outline:hover { background: var(--green-light); }
.btn-orange { background: var(--orange); border-color: var(--orange); }
.btn-orange:hover { background: #ea580c; }
.btn-danger { background: var(--red); border-color: var(--red); }
.btn-danger:hover { background: #b91c1c; }
.btn-amber { background: var(--amber); border-color: var(--amber); }
.btn-amber:hover { background: #b45309; }
.btn-slate { background: var(--slate-soft); border-color: var(--slate-soft); }
.btn-slate:hover { background: #0f172a; }
.btn-row { display: flex; gap: 10px; flex-wrap: wrap; align-items: center; }
.inline { display: inline; }
.inline-form { display: inline; margin: 0; }

/* -------------------------------- cards --------------------------------- */
.card {
    background: var(--card);
    border: 1px solid var(--border);
    border-radius: var(--radius);
    box-shadow: var(--shadow);
    padding: 20px;
    margin-bottom: 20px;
}
.card.tight { padding: 14px 16px; }
.card-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(265px, 1fr));
    gap: 18px;
    margin-bottom: 22px;
}
.page-head {
    display: flex;
    justify-content: space-between;
    align-items: flex-end;
    gap: 14px;
    flex-wrap: wrap;
    margin-bottom: 16px;
}
.page-head p { margin: 0; }

/* ------------------------------- hero ----------------------------------- */
.hero {
    background: linear-gradient(120deg, var(--slate) 0%, #14532d 60%, var(--green-dark) 100%);
    color: #fff;
    border-radius: var(--radius);
    padding: 34px 30px;
    margin-bottom: 22px;
    box-shadow: var(--shadow);
}
.hero h1 { color: #fff; font-size: 2rem; margin-bottom: 8px; }
.hero p { color: #dcfce7; margin: 0 0 18px; max-width: 640px; }

/* ------------------------------ stat cards ------------------------------ */
.stats-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(170px, 1fr));
    gap: 14px;
    margin-bottom: 24px;
}
.stat-card {
    background: var(--card);
    border: 1px solid var(--border);
    border-radius: var(--radius);
    box-shadow: var(--shadow);
    padding: 16px 18px;
}
.stat-card .stat-value { font-size: 1.6rem; font-weight: 800; color: var(--slate); }
.stat-card .stat-label { color: var(--muted); font-size: .85rem; }
.stat-card.accent-green { border-left: 4px solid var(--green); }
.stat-card.accent-orange { border-left: 4px solid var(--orange); }
.stat-card.accent-blue { border-left: 4px solid var(--blue); }
.stat-card.accent-amber { border-left: 4px solid var(--amber); }

/* -------------------------------- tables -------------------------------- */
.table-wrap { overflow-x: auto; }
table.table {
    width: 100%;
    border-collapse: collapse;
    background: var(--card);
    border-radius: 12px;
    overflow: hidden;
    box-shadow: var(--shadow);
    font-size: .92rem;
}
.table th {
    background: var(--slate-soft);
    color: #e2e8f0;
    text-align: left;
    padding: 11px 14px;
    font-size: .82rem;
    text-transform: uppercase;
    letter-spacing: .4px;
}
.table td { padding: 11px 14px; border-bottom: 1px solid var(--border); vertical-align: middle; }
.table tr:last-child td { border-bottom: none; }
.table tbody tr:hover { background: #f8fafc; }
.table .actions { white-space: nowrap; display: flex; gap: 6px; flex-wrap: wrap; }

/* -------------------------------- badges -------------------------------- */
.badge {
    display: inline-block;
    padding: 3px 10px;
    border-radius: 999px;
    font-size: .76rem;
    font-weight: 700;
    border: 1px solid transparent;
}
.status-placed { background: var(--blue-light); color: #1e40af; border-color: #93c5fd; }
.status-confirmed { background: #e0e7ff; color: #3730a3; border-color: #a5b4fc; }
.status-preparing { background: var(--amber-light); color: #92400e; border-color: #fcd34d; }
.status-out-for-delivery { background: var(--orange-light); color: #9a3412; border-color: #fdba74; }
.status-delivered { background: var(--green-light); color: #166534; border-color: #86efac; }
.status-cancelled { background: var(--red-light); color: #991b1b; border-color: #fca5a5; }
.status-pending { background: var(--amber-light); color: #92400e; border-color: #fcd34d; }
.status-paid { background: var(--green-light); color: #166534; border-color: #86efac; }
.status-failed { background: var(--red-light); color: #991b1b; border-color: #fca5a5; }
.status-refunded { background: #e2e8f0; color: #334155; border-color: #cbd5e1; }
.status-available { background: var(--green-light); color: #166534; border-color: #86efac; }
.status-on-delivery { background: var(--orange-light); color: #9a3412; border-color: #fdba74; }
.status-offline { background: #e2e8f0; color: #334155; border-color: #cbd5e1; }
.chip {
    display: inline-block;
    background: var(--bg);
    border: 1px solid var(--border);
    border-radius: 999px;
    padding: 2px 10px;
    font-size: .76rem;
    color: var(--muted);
}
.chip-veg { background: var(--green-light); color: #166534; border-color: #86efac; }

/* ------------------------------ menu cards ------------------------------ */
.dish-card { display: flex; flex-direction: column; }
.dish-card .dish-head { display: flex; justify-content: space-between; gap: 10px; align-items: flex-start; }
.dish-card .price { font-weight: 800; color: var(--green-dark); font-size: 1.05rem; white-space: nowrap; }
.dish-card p { color: var(--muted); font-size: .88rem; margin: 6px 0 10px; flex: 1; }
.dish-card .dish-actions { display: flex; gap: 8px; align-items: center; }
.qty-input {
    width: 62px;
    padding: 7px 8px;
    border: 1px solid var(--border);
    border-radius: 8px;
    font-size: .9rem;
}
.stars { color: #f59e0b; letter-spacing: 2px; font-size: 1rem; }

/* -------------------------------- forms --------------------------------- */
.form-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 14px 18px; }
.form-grid .full { grid-column: 1 / -1; }
@media (max-width: 700px) { .form-grid { grid-template-columns: 1fr; } }
.form-group label { display: block; font-weight: 600; font-size: .88rem; margin-bottom: 5px; color: var(--slate); }
.form-group input,
.form-group select,
.form-group textarea {
    width: 100%;
    padding: 9px 12px;
    border: 1px solid var(--border);
    border-radius: 10px;
    font-size: .93rem;
    font-family: inherit;
    background: #fff;
    color: var(--text);
}
.form-group input:focus,
.form-group select:focus,
.form-group textarea:focus {
    outline: 2px solid var(--green);
    outline-offset: 1px;
    border-color: var(--green);
}
.form-group textarea { resize: vertical; min-height: 84px; }
.check-row { display: flex; align-items: center; gap: 8px; margin-top: 26px; }
.check-row input { width: auto; }
.form-actions { margin-top: 18px; display: flex; gap: 10px; flex-wrap: wrap; }
.filter-form {
    display: flex;
    gap: 10px;
    align-items: flex-end;
    flex-wrap: wrap;
    margin-bottom: 16px;
}
.filter-form .form-group { margin: 0; min-width: 200px; }

/* ------------------------------- totals --------------------------------- */
.totals { max-width: 420px; margin-left: auto; }
.totals .row { display: flex; justify-content: space-between; padding: 7px 0; border-bottom: 1px dashed var(--border); }
.totals .row:last-child { border-bottom: none; font-weight: 800; font-size: 1.1rem; color: var(--slate); }
.totals .grand { color: var(--green-dark); }

/* ------------------------------ empty state ----------------------------- */
.empty {
    text-align: center;
    padding: 44px 20px;
    color: var(--muted);
    background: var(--card);
    border: 1px dashed var(--border);
    border-radius: var(--radius);
}
.empty .big { font-size: 2.4rem; display: block; margin-bottom: 8px; }

/* -------------------------------- footer -------------------------------- */
.footer {
    background: var(--slate);
    color: #94a3b8;
    padding: 22px 0;
    margin-top: 30px;
    font-size: .88rem;
}
.footer p { margin: 4px 0; }
.footer .muted { color: #64748b; }

/* ------------------------------ misc bits ------------------------------- */
.avatar {
    width: 44px;
    height: 44px;
    border-radius: 50%;
    background: var(--green-light);
    color: var(--green-dark);
    display: inline-flex;
    align-items: center;
    justify-content: center;
    font-weight: 800;
    font-size: 1rem;
}
.kv { display: grid; grid-template-columns: 170px 1fr; gap: 6px 14px; font-size: .93rem; }
.kv dt { color: var(--muted); }
.kv dd { margin: 0; font-weight: 600; }
.section-title {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin: 26px 0 12px;
}
.section-title h2 { margin: 0; }
.restaurant-head {
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 12px;
    flex-wrap: wrap;
    border-bottom: 2px solid var(--border);
    padding-bottom: 8px;
    margin-bottom: 14px;
}

```
