# Week-3-Day-5-Stripe-Payment-Verification
Right now, a customer can return to /payment-success, but we haven't actually verified with Stripe that the payment succeeded.

1. Update paymentController.js

Open:

backend/controllers/paymentController.js

Keep your existing createCheckoutSession function.

Then add this function below it:

const verifyCheckoutSession = async (req, res) => {
  try {
    if (req.user.role !== "customer") {
      return res.status(403).json({
        message: "Only customers can verify payments"
      });
    }

    const { sessionId } = req.body;

    if (!sessionId) {
      return res.status(400).json({
        message: "Stripe session ID is required"
      });
    }

    // Get session from Stripe
    const session =
      await stripe.checkout.sessions.retrieve(
        sessionId
      );

    // Make sure this session belongs to the customer
    if (
      session.metadata?.customerId !==
      req.user.userId.toString()
    ) {
      return res.status(403).json({
        message: "This payment does not belong to you"
      });
    }

    // Check payment status
    if (session.payment_status !== "paid") {
      return res.status(400).json({
        message: "Payment has not been completed",
        paymentStatus:
          session.payment_status
      });
    }

    res.status(200).json({
      message: "Payment verified successfully",

      paymentStatus:
        session.payment_status,

      sessionId:
        session.id,

      paymentIntentId:
        session.payment_intent,

      customerId:
        session.metadata.customerId,

      storeId:
        session.metadata.storeId
    });

  } catch (error) {

    console.error(
      "Payment verification error:",
      error
    );

    res.status(500).json({
      message: error.message
    });
  }
};

Then change the export at the bottom:

module.exports = {
  createCheckoutSession,
  verifyCheckoutSession
};
2. Add Verification Route

Open:

backend/routes/paymentRoutes.js

Change the controller import:

const {
  createCheckoutSession,
  verifyCheckoutSession
} = require("../controllers/paymentController");

Then add:

router.post(
  "/verify-session",
  protect,
  authorizeRoles("customer"),
  verifyCheckoutSession
);

Your payment routes are now:

POST /api/payments/create-checkout-session/:storeId
POST /api/payments/verify-session
3. Test the Verification

After Stripe redirects the customer back, the URL will contain:

/payment-success?session_id=cs_test_...

We'll read that session ID in React.

Open:

frontend/src/pages/PaymentSuccess.jsx

Replace the existing code with:

import { useEffect, useState } from "react";
import { Link, useSearchParams } from "react-router-dom";

import API from "../services/api";
import { useDispatch } from "react-redux";
import { clearCart } from "../redux/cartSlice";

function PaymentSuccess() {

  const [searchParams] =
    useSearchParams();

  const dispatch = useDispatch();

  const [loading, setLoading] =
    useState(true);

  const [verified, setVerified] =
    useState(false);

  const [message, setMessage] =
    useState("");

  const verifyPayment = async () => {

    try {

      const sessionId =
        searchParams.get("session_id");

      if (!sessionId) {
        setMessage(
          "Payment session not found"
        );

        setLoading(false);
        return;
      }

      const response =
        await API.post(
          "/payments/verify-session",
          {
            sessionId
          }
        );

      if (
        response.data.paymentStatus ===
        "paid"
      ) {

        setVerified(true);

        setMessage(
          "Payment verified successfully!"
        );

        dispatch(clearCart());

      } else {

        setMessage(
          "Payment could not be verified"
        );
      }

    } catch (error) {

      setMessage(
        error.response?.data?.message ||
        "Unable to verify payment"
      );

    } finally {

      setLoading(false);
    }
  };


  useEffect(() => {
    verifyPayment();
  }, []);


  if (loading) {

    return (

      <div className="order-success">

        <h1>
          Verifying Payment...
        </h1>

        <p>
          Please wait while we verify
          your Stripe payment.
        </p>

      </div>

    );
  }


  return (

    <div className="order-success">

      {verified ? (

        <>
          <h1>
            Payment Verified! 🎉
          </h1>

          <p>
            {message}
          </p>

          <p>
            Your payment was successfully
            verified by our server.
          </p>

          <Link to="/shop">
            Continue Shopping
          </Link>
        </>

      ) : (

        <>
          <h1>
            Payment Verification Failed
          </h1>

          <p>
            {message}
          </p>

          <Link to="/checkout">
            Return to Checkout
          </Link>
        </>

      )}

    </div>

  );
}

export default PaymentSuccess;
4. What Happens Now?

When Stripe sends the customer back:

/payment-success?session_id=cs_test_123

React gets:

const sessionId =
  searchParams.get("session_id");

Then sends it to:

POST /api/payments/verify-session

The backend asks Stripe:

"Is this Checkout Session actually paid?"

Stripe responds with:

payment_status = paid

Only then do we display:

Payment Verified! 🎉
5. Important Problem With Our Current Order Flow

There's one thing we need to fix.

Currently Day 3 created orders before payment:

Cart
 ↓
Create Order
 ↓
Payment

But now we want:

Cart
 ↓
Payment
 ↓
Verify Payment
 ↓
Create Order

This is better because your database shouldn't mark an unpaid purchase as a completed order.

So we'll modify the order process.

6. Create createOrderFromPayment

Open:

backend/controllers/orderController.js

Add this function:

const createOrderFromPayment = async (
  req,
  res
) => {

  try {

    if (req.user.role !== "customer") {
      return res.status(403).json({
        message:
          "Only customers can create orders"
      });
    }

    const {
      storeId,
      paymentIntentId
    } = req.body;


    if (!storeId || !paymentIntentId) {
      return res.status(400).json({
        message:
          "Store ID and payment ID are required"
      });
    }


    // Find customer's cart
    const cart = await Cart.findOne({
      customerId: req.user.userId,
      storeId: storeId
    });


    if (
      !cart ||
      cart.items.length === 0
    ) {

      return res.status(400).json({
        message: "Your cart is empty"
      });

    }


    const orderItems = [];

    let totalAmount = 0;


    // Validate products again
    for (const cartItem of cart.items) {

      const product =
        await Product.findOne({
          _id: cartItem.productId,
          storeId: storeId
        });


      if (!product) {

        return res.status(400).json({
          message:
            `${cartItem.name} is no longer available`
        });

      }


      if (
        product.stock <
        cartItem.quantity
      ) {

        return res.status(400).json({
          message:
            `${product.name} has only ${product.stock} items available`
        });

      }


      orderItems.push({

        productId:
          product._id,

        name:
          product.name,

        quantity:
          cartItem.quantity,

        price:
          product.price,

        image:
          product.image

      });


      totalAmount +=
        product.price *
        cartItem.quantity;

    }


    // Create order
    const order =
      await Order.create({

        customerId:
          req.user.userId,

        storeId:

          storeId,

        items:
          orderItems,

        totalAmount:
          totalAmount,

        paymentStatus:
          "paid",

        orderStatus:
          "placed",

        stripePaymentId:
          paymentIntentId

      });


    // Reduce product stock
    for (const item of orderItems) {

      await Product.findByIdAndUpdate(
        item.productId,

        {
          $inc: {
            stock:
              -item.quantity
          }
        }
      );

    }


    // Clear cart
    cart.items = [];

    cart.totalAmount = 0;

    await cart.save();


    res.status(201).json({

      message:
        "Order created successfully",

      order

    });


  } catch (error) {

    res.status(500).json({
      message: error.message
    });

  }

};

At the top of orderController.js, make sure you have:

const Order = require("../models/Order");
const Cart = require("../models/Cart");
const Product = require("../models/Product");
const Store = require("../models/Store");

Then export it:

module.exports = {
  createOrder,
  getMyOrders,
  createOrderFromPayment
};
7. Add Route

Open:

backend/routes/orderRoutes.js

Change the import:

const {
  createOrder,
  getMyOrders,
  createOrderFromPayment
} = require("../controllers/orderController");

Add:

router.post(
  "/from-payment",
  protect,
  authorizeRoles("customer"),
  createOrderFromPayment
);

Now:

POST /api/orders/from-payment

will create the final paid order.

8. Update Payment Success Page Again

Now, after verifying Stripe, we will create the order.

Inside verifyPayment(), after:

if (
  response.data.paymentStatus ===
  "paid"
) {

we'll call the order API.

Replace that section with:

if (
  response.data.paymentStatus ===
  "paid"
) {

  const orderResponse =
    await API.post(
      "/orders/from-payment",
      {
        storeId:
          response.data.storeId,

        paymentIntentId:
          response.data.paymentIntentId
      }
    );

  setVerified(true);

  setMessage(
    "Payment verified and order created successfully!"
  );

  dispatch(clearCart());

  console.log(
    "Created Order:",
    orderResponse.data.order
  );

}
9. Final Payment Flow

Now your project has:

                 CUSTOMER
                    │
                    ▼
                  SHOP
                    │
                    ▼
                  CART
                    │
                    ▼
                CHECKOUT
                    │
                    ▼
             Create Stripe
               Session
                    │
                    ▼
             Stripe Checkout
                    │
                    ▼
               Pay ₹₹₹
                    │
                    ▼
           Stripe Success URL
                    │
                    ▼
          Get session_id
                    │
                    ▼
        Backend verifies Stripe
                    │
              ┌─────┴─────┐
              │           │
             FAIL        PAID
              │           │
              ▼           ▼
          Show error   Create Order
                          │
                          ▼
                     Reduce Stock
                          │
                          ▼
                      Clear Cart
                          │
                          ▼
                   Order Created
10. Prevent Duplicate Orders

There's another important issue.

The customer could refresh:

/payment-success?session_id=...

and potentially create the order twice.

We should prevent that.

Before creating an order, check whether that Stripe PaymentIntent has already been used.

Inside createOrderFromPayment, add this before creating the order:

const existingOrder =
  await Order.findOne({
    stripePaymentId:
      paymentIntentId
  });

if (existingOrder) {

  return res.status(200).json({
    message:
      "Order already exists",

    order:
      existingOrder
  });

}

Now:

First request
     ↓
Payment ID not found
     ↓
Create order ✅

Second request
     ↓
Payment ID already exists
     ↓
Return existing order
     ↓
No duplicate order

That's a very useful protection for your project.

11. Your Order Document Now Looks Like

After successful payment:

{
  "customerId": "customer_id",
  "storeId": "store_id",

  "items": [
    {
      "productId": "product_id",
      "name": "T-Shirt",
      "quantity": 2,
      "price": 500,
      "image": "cloudinary_url"
    }
  ],

  "totalAmount": 1000,

  "paymentStatus": "paid",

  "orderStatus": "placed",

  "stripePaymentId": "pi_xxxxxxxxx"
}

This is much better than the earlier:

paymentStatus: pending

because the order is created after successful payment verification.

12. Test Everything

Start backend:

cd backend
node server.js

Start frontend:

cd frontend
npm run dev

Then:

Login
 ↓
Shop
 ↓
Add Product
 ↓
Cart
 ↓
Checkout
 ↓
Pay with Stripe
 ↓
Stripe Test Payment
 ↓
Payment Success
 ↓
Payment Verification
 ↓
Order Creation
 ↓
Stock Reduction
 ↓
Cart Cleared

Check MongoDB:

products
   ↓
stock reduced

carts
   ↓
empty

orders
   ↓
new order

paymentStatus
   ↓
paid

orderStatus
   ↓
placed
