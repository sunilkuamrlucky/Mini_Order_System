# Mini_Order_System
Creating a mini orders system for Food Suppliers to easy to Handle or use. 
To build a RESTful API using FastAPI that allows users to:

1. Create customer orders with product items.

2. View all orders and individual orders.

3. Update the order status (Pending, Processing, Completed, Cancelled).
4. Delete orders when not needed.

5. Calculate totals including:
Subtotal (before discount & tax)
Discount amount
Tax amount
Grand total

6. Get order summary, including:
Total number of orders
Total value of all orders

Target Users / Stakeholders:

Small businesses or e-commerce platforms
Backend developers or analytics teams
Interns learning API development and data handling

Key Requirements (Functional):

Order Creation: Add a Customer name, multiple items, tax and discount
Order Retrieval: Get all orders or a single order by ID
Order Update: Change order status (PATCH endpoint)
Order Deletion: Remove orders when no longer needed
Order Summary: Display counts and total values, grouped by status

Non-Functional Requirements:
Use FastAPI (Python) for building the API
Store orders in in-memory DB (orders_db dictionary)
Return JSON responses for easy integration with frontend or analytics tools
Include Swagger docs at /docs for testing endpoints

The we used  python liabraries are below 

To install that liabraries use - pip install fastapi uvicorn

to run the code use - uvicorn orders:app --reload

To run the application:

1. Save the code as main.py (or any other name).

2. Open your terminal or command prompt in the same directory.
   
3. Install FastAPI and Uvicorn: pip install fastapi uvicorn pydantic
   
4. Run the application: uvicorn main:app --reload
 
5. Access the API documentation (Swagger UI) at: http://127.0.0.1:8000/docs
   
6. Access the alternative documentation (ReDoc) at: http://127.0.0.1:8000/redoc

you get below options such as:

1. GET / Read Root: This is typically a very basic endpoint that serves as a health check or a simple welcome message for your API. In a FastAPI application, if you define an endpoint like @app.get("/"), this is what it corresponds to. When you access http://127.0.0.1:8000/ (or wherever your API is hosted) directly in your browser, this endpoint is usually hit. It's often used to confirm that the server is running.

2. GET /orders/ Get All Orders: This endpoint is designed to retrieve a collection of all the orders that are currently stored in your system. When a client sends a GET request to /orders/, the API will respond with a list of all existing order records. This is useful for an administrator or a system that needs to view the entire order history.
3. POST /orders/ Create Order: This is the endpoint used to create a new order. When a client (like a mobile app or a web frontend) wants to place an order, it will send a POST request to /orders/. This request will include the details of the new order (e.g., customer name, items, quantities, discount, tax) in its "request body." The API then processes this data, creates a new order record, stores it (in your case, in memory), and typically responds with the details of the newly created order, often including a unique order_id.

4. GET /orders/{order_id} Get Order Summary (Likely "Get Single Order"): This endpoint is used to fetch a specific order by its unique identifier (order_id). The {order_id} part in the path signifies a "path parameter," meaning the client needs to replace this placeholder with an actual order's ID (e.g., /orders/abcdef123). The API will look up the order corresponding to that ID and return its details. The title "Get Order Summary" might be a slight misnomer in the image, as a GET on /orders/{order_id} typically returns the full details of that single order, while "summary" usually refers to an aggregation of all orders (like total count, total value), which is handled by the GET /orders/summary/ endpoint in your full code.

5. DELETE /orders/{order_id} Delete Order: This endpoint is used to remove (delete) a specific order from the system. Similar to the GET endpoint for a single order, it uses the order_id as a path parameter. When a DELETE request is sent to this path, the API attempts to find and remove the specified order. If successful, it usually returns a 204 No Content status code, indicating that the operation was successful but there's no content to return.

6. PATCH /orders/{order_id}/status Update Order Status: This endpoint is specifically designed to partially update an existing order, specifically its status field. PATCH requests are used for partial modifications. The order_id identifies which order to update, and the request body for this endpoint would likely contain just the new status (e.g., "processing", "completed", "cancelled"). This allows for modifying only a specific attribute of an order without sending the entire order object.

```from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Optional
from enum import Enum
from uuid import uuid4
from datetime import datetime

app = FastAPI()

# Enum for order status
class OrderStatus(str, Enum):
    PENDING = "pending"
    PROCESSING = "processing"
    COMPLETED = "completed"
    CANCELLED = "cancelled"

# Model for order items
class OrderItem(BaseModel):
    product_id: str
    name: str
    quantity: int
    price: float

# Model for creating an order
class OrderCreate(BaseModel):
    customer_name: str
    items: List[OrderItem]
    discount: Optional[float] = 0.0  # Discount percentage (e.g., 10 for 10%)
    tax_rate: Optional[float] = 0.0  # Tax percentage (e.g., 8 for 8%)

# Model for order response
class OrderResponse(OrderCreate):
    id: str
    status: OrderStatus
    created_at: datetime
    updated_at: datetime
    subtotal: float
    discount_amount: float
    tax_amount: float
    total: float

# In-memory storage
orders_db = {}

# Helper function to calculate order totals
def calculate_order_totals(items: List[OrderItem], discount: float, tax_rate: float) -> dict:
    subtotal = sum(item.price * item.quantity for item in items)
    discount_amount = subtotal * (discount / 100)
    subtotal_after_discount = subtotal - discount_amount
    tax_amount = subtotal_after_discount * (tax_rate / 100)
    total = subtotal_after_discount + tax_amount
    
    return {
        "subtotal": round(subtotal, 2),
        "discount_amount": round(discount_amount, 2),
        "tax_amount": round(tax_amount, 2),
        "total": round(total, 2)
    }

@app.get("/")
def read_root():
    return {"message": "FastAPI is running . Visit /docs for API Documenatation."}

# Create a new order
@app.post("/orders/", response_model=OrderResponse)
async def create_order(order: OrderCreate):
    order_id = str(uuid4())
    now = datetime.now()
    
    totals = calculate_order_totals(order.items, order.discount, order.tax_rate)
    
    order_data = {
        "id": order_id,
        "customer_name": order.customer_name,
        "items": [item.dict() for item in order.items],
        "status": OrderStatus.COMPLETED,
        "created_at": now,
        "updated_at": now,
        "discount": order.discount,
        "tax_rate": order.tax_rate,
        **totals
    }
    
    orders_db[order_id] = order_data
    return order_data

# Get all orders
@app.get("/orders/", response_model=List[OrderResponse])
async def get_all_orders():
    return list(orders_db.values())

# Get a single order by ID
@app.get("/orders/{order_id}", response_model=OrderResponse)
async def get_order(order_id: str):
    if order_id not in orders_db:
        raise HTTPException(status_code=404, detail="Order not found")
    return orders_db[order_id]

# Update order status
@app.patch("/orders/{order_id}/status", response_model=OrderResponse)
async def update_order_status(order_id: str, status: OrderStatus):
    if order_id not in orders_db:
        raise HTTPException(status_code=404, detail="Order not found")
    
    orders_db[order_id]["status"] = status
    orders_db[order_id]["updated_at"] = datetime.now()
    
    return orders_db[order_id]

# Delete an order by ID
@app.delete("/orders/{order_id}", status_code=204)
async def delete_order(order_id: str):
    if order_id not in orders_db:
        raise HTTPException(status_code=404, detail="Order not found")
    del orders_db[order_id]
    return

# Get order summary
@app.get("/orders/{order_id}", response_model=OrderResponse)
async def get_order_summary(order_id: str):
    total_orders = len(orders_db)
    total_value = sum(order["total"] for order in orders_db.values())
    total_pending = sum(1 for order in orders_db.values() if order["status"] == OrderStatus.PENDING)
    total_processing = sum(1 for order in orders_db.values() if order["status"] == OrderStatus.PROCESSING)
    total_completed = sum(1 for order in orders_db.values() if order["status"] == OrderStatus.COMPLETED)
    total_cancelled = sum(1 for order in orders_db.values() if order["status"] == OrderStatus.CANCELLED)
    
    return {
        "total_orders": total_orders,
        "total_value": round(total_value, 2),
        "status_counts": {
            "pending": total_pending,
            "processing": total_processing,
            "completed": total_completed,
            "cancelled": total_cancelled
        }
    }

 
 




