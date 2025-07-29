## Problematic Ruby/Rails code snippets
**
Comprehensive collection of problematic Ruby/Rails code snippets that are commonly used. These cover the key areas mentioned and will help practice identifying issues and explaining improvements.

**Key Areas Covered:**
- **Performance Issues**: N+1 queries, memory inefficient processing, poor caching
- **Security Vulnerabilities**: Mass assignment, SQL injection, information disclosure, missing authorization
- **Design Pattern Violations**: Fat models, god controllers, single responsibility violations
- **Testing Gaps**: Untestable code with hard dependencies, missing edge case handling
- **Maintainability Problems**: Complex conditionals, magic numbers, deeply nested logic



# Ruby/Rails Interview Code Review Practice
---

## 1. Performance Issue - N+1 Query Problem

**Problem:** Classic N+1 query issue
```ruby
class PostsController < ApplicationController
  def index
    @posts = Post.all
    @posts.each do |post|
      puts "Post: #{post.title} by #{post.user.name}"
      puts "Comments: #{post.comments.count}"
    end
  end
end
```

**Issues to Identify:**
- N+1 queries for user and comments
- No pagination for potentially large datasets
- Business logic in controller

---

## 2. Security Vulnerability - Mass Assignment

**Problem:** Unsafe parameter handling
```ruby
class UsersController < ApplicationController
  def create
    @user = User.new(params[:user])
    if @user.save
      redirect_to @user
    else
      render :new
    end
  end

  def update
    @user = User.find(params[:id])
    @user.update_attributes(params[:user])
    redirect_to @user
  end
end
```

**Issues to Identify:**
- Mass assignment vulnerability
- Missing strong parameters
- No authorization checks
- Missing error handling

---

## 3. Design Pattern Violation - Fat Model

**Problem:** Single Responsibility Principle violation
```ruby
class User < ActiveRecord::Base
  def full_name
    "#{first_name} #{last_name}"
  end

  def send_welcome_email
    UserMailer.welcome_email(self).deliver_now
  end

  def calculate_total_spent
    orders.sum(:total_amount)
  end

  def generate_report
    report = {}
    report[:total_orders] = orders.count
    report[:total_spent] = calculate_total_spent
    report[:favorite_category] = orders.joins(:items).group('items.category').count.max_by{|k,v| v}.first
    
    # Generate PDF
    pdf = Prawn::Document.new
    pdf.text("User Report for #{full_name}")
    pdf.text("Total Orders: #{report[:total_orders]}")
    pdf.text("Total Spent: $#{report[:total_spent]}")
    pdf.render
  end

  def sync_with_external_service
    response = HTTParty.get("https://api.example.com/users/#{id}")
    if response.success?
      update(external_id: response['id'])
    end
  end
end
```

**Issues to Identify:**
- Too many responsibilities in one class
- External API calls in model
- PDF generation logic in model
- Missing service objects

---

## 4. Performance Issue - Inefficient Database Queries

**Problem:** Multiple inefficient queries and operations
```ruby
class OrderAnalyzer
  def self.monthly_report(year, month)
    orders = Order.all
    monthly_orders = []
    
    orders.each do |order|
      if order.created_at.year == year && order.created_at.month == month
        monthly_orders << order
      end
    end
    
    total_revenue = 0
    monthly_orders.each do |order|
      total_revenue += order.total_amount
    end
    
    # Find top customer
    customer_orders = {}
    monthly_orders.each do |order|
      customer_orders[order.user_id] ||= 0
      customer_orders[order.user_id] += order.total_amount
    end
    
    top_customer_id = customer_orders.max_by { |k, v| v }.first
    top_customer = User.find(top_customer_id)
    
    {
      total_orders: monthly_orders.count,
      total_revenue: total_revenue,
      top_customer: top_customer.full_name
    }
  end
end
```

**Issues to Identify:**
- Loading all orders into memory
- Ruby-based filtering instead of SQL
- Multiple loops instead of single query
- No database indexes consideration

---

## 5. Security Vulnerability - SQL Injection

**Problem:** Unsafe SQL construction
```ruby
class ProductSearch
  def self.search(category, price_range, name)
    query = "SELECT * FROM products WHERE 1=1"
    
    if category.present?
      query += " AND category = '#{category}'"
    end
    
    if price_range.present?
      query += " AND price BETWEEN #{price_range[:min]} AND #{price_range[:max]}"
    end
    
    if name.present?
      query += " AND name LIKE '%#{name}%'"
    end
    
    ActiveRecord::Base.connection.execute(query)
  end
end
```

**Issues to Identify:**
- SQL injection vulnerability
- No parameter sanitization
- Raw SQL instead of ActiveRecord
- No input validation

---

## 6. Testing Gap - Untestable Code

**Problem:** Code that's difficult to test
```ruby
class PaymentProcessor
  def process_payment(order)
    # Get current time
    timestamp = Time.now
    
    # Call external API
    response = HTTParty.post('https://payment-gateway.com/charge', {
      body: {
        amount: order.total_amount,
        card_token: order.payment_token,
        timestamp: timestamp
      }
    })
    
    if response.code == 200
      order.update(
        status: 'paid',
        paid_at: timestamp,
        transaction_id: response['transaction_id']
      )
      
      # Send email
      PaymentMailer.payment_confirmed(order).deliver_now
      
      # Log to file
      File.open('/var/log/payments.log', 'a') do |f|
        f.write("#{timestamp}: Payment processed for order #{order.id}\n")
      end
      
      true
    else
      false
    end
  end
end
```

**Issues to Identify:**
- Hard dependencies on external services
- File I/O in business logic
- No dependency injection
- Mixed responsibilities
- Hard to mock/stub

---

## 7. Maintainability Problem - Complex Conditionals

**Problem:** Deeply nested conditions and complex logic
```ruby
class DiscountCalculator
  def calculate_discount(user, items, promo_code)
    total = items.sum(&:price)
    discount = 0
    
    if user.premium?
      if user.subscription_active?
        if total > 100
          discount = total * 0.15
        elsif total > 50
          discount = total * 0.10
        else
          discount = total * 0.05
        end
      else
        if total > 100
          discount = total * 0.10
        elsif total > 50
          discount = total * 0.05
        end
      end
    else
      if promo_code == 'SAVE10'
        if user.first_time_buyer?
          if total > 50
            discount = total * 0.10
          else
            discount = 5
          end
        else
          discount = total * 0.05
        end
      elsif promo_code == 'WELCOME'
        if user.first_time_buyer?
          discount = [total * 0.20, 25].min
        end
      end
    end
    
    discount
  end
end
```

**Issues to Identify:**
- Deeply nested conditionals
- Complex business logic in single method
- Hard to understand and maintain
- No clear pattern or strategy

---

## 8. Performance Issue - Memory Inefficient Processing

**Problem:** Loading large datasets into memory
```ruby
class DataExporter
  def export_user_data
    users = User.all.includes(:orders, :profile)
    csv_data = []
    
    users.each do |user|
      user.orders.each do |order|
        csv_data << [
          user.id,
          user.email,
          user.full_name,
          order.id,
          order.total_amount,
          order.created_at.strftime('%Y-%m-%d')
        ]
      end
    end
    
    CSV.generate do |csv|
      csv << ['User ID', 'Email', 'Name', 'Order ID', 'Amount', 'Date']
      csv_data.each { |row| csv << row }
    end
  end
end
```

**Issues to Identify:**
- Loading all users into memory
- Building large array in memory
- No batch processing
- Potential memory overflow

---

## 9. Security Vulnerability - Information Disclosure

**Problem:** Exposing sensitive information
```ruby
class UsersController < ApplicationController
  def show
    @user = User.find(params[:id])
    render json: @user.as_json
  end

  def index
    @users = User.all
    render json: @users.as_json(include: [:orders, :payment_methods])
  end
end

class User < ActiveRecord::Base
  def as_json(options = {})
    super(options.merge(
      include: [:profile, :orders],
      methods: [:full_name]
    ))
  end
end
```

**Issues to Identify:**
- Exposing all user attributes including sensitive ones
- No authorization checks
- Including sensitive associations
- No API versioning or serialization control

---

## 10. Design Pattern Violation - God Class Controller

**Problem:** Controller doing too much
```ruby
class OrdersController < ApplicationController
  def create
    # Validate inventory
    params[:order][:items].each do |item_data|
      item = Item.find(item_data[:id])
      if item.quantity < item_data[:quantity]
        flash[:error] = "Not enough inventory for #{item.name}"
        redirect_to cart_path and return
      end
    end
    
    # Calculate total
    total = 0
    params[:order][:items].each do |item_data|
      item = Item.find(item_data[:id])
      total += item.price * item_data[:quantity]
    end
    
    # Apply discount
    if params[:promo_code].present?
      case params[:promo_code]
      when 'SAVE10'
        total *= 0.9
      when 'SAVE20'
        total *= 0.8
      end
    end
    
    # Process payment
    payment_response = HTTParty.post('https://payments.com/charge', {
      body: {
        amount: total,
        token: params[:payment_token]
      }
    })
    
    if payment_response.success?
      # Create order
      @order = Order.create!(
        user: current_user,
        total_amount: total,
        status: 'processing'
      )
      
      # Create order items
      params[:order][:items].each do |item_data|
        item = Item.find(item_data[:id])
        @order.order_items.create!(
          item: item,
          quantity: item_data[:quantity],
          price: item.price
        )
        
        # Update inventory
        item.update!(quantity: item.quantity - item_data[:quantity])
      end
      
      # Send confirmation email
      OrderMailer.confirmation(@order).deliver_now
      
      redirect_to @order
    else
      flash[:error] = "Payment failed"
      redirect_to cart_path
    end
  end
end
```

**Issues to Identify:**
- Multiple responsibilities in controller
- Business logic in controller
- External API calls in controller
- No service objects or form objects
- Poor error handling

---

## 11. Performance Issue - Inefficient Caching

**Problem:** Poor caching strategy
```ruby
class ProductsController < ApplicationController
  def index
    @categories = Rails.cache.fetch('categories') do
      Category.all.to_a
    end
    
    @featured_products = Rails.cache.fetch('featured_products') do
      Product.where(featured: true).limit(10).to_a
    end
    
    if params[:category_id]
      @products = Product.where(category_id: params[:category_id])
    else
      @products = Product.all
    end
    
    @products.each do |product|
      product.view_count += 1
      product.save!
    end
  end
end
```

**Issues to Identify:**
- Inefficient cache keys (no expiration strategy)
- N+1 queries for updating view counts
- No conditional caching based on parameters
- Synchronous counter updates

---

## 12. Testing Gap - Missing Edge Cases

**Problem:** Incomplete validation and error handling
```ruby
class BankAccount
  attr_accessor :balance

  def initialize(initial_balance)
    @balance = initial_balance
  end

  def withdraw(amount)
    @balance -= amount
  end

  def deposit(amount)
    @balance += amount
  end

  def transfer(amount, target_account)
    withdraw(amount)
    target_account.deposit(amount)
  end
end
```

**Issues to Identify:**
- No input validation
- No balance checks for withdrawals
- No error handling for edge cases
- No thread safety
- Missing business rule enforcement

---

## 13. Maintainability Problem - Magic Numbers and Strings

**Problem:** Hard-coded values throughout code
```ruby
class SubscriptionManager
  def calculate_price(plan_type, duration)
    case plan_type
    when 'basic'
      base_price = 9.99
    when 'premium'
      base_price = 19.99
    when 'enterprise'
      base_price = 49.99
    end
    
    case duration
    when 1
      discount = 0
    when 6
      discount = 0.05
    when 12
      discount = 0.15
    end
    
    total = base_price * duration * (1 - discount)
    
    if total > 100
      total *= 0.95  # Volume discount
    end
    
    total
  end

  def send_renewal_notice(subscription)
    if subscription.expires_at - Time.now < 86400 * 7  # 7 days
      SubscriptionMailer.renewal_notice(subscription).deliver_now
    end
  end
end
```

**Issues to Identify:**
- Magic numbers scattered throughout
- Hard-coded pricing logic
- No configuration management
- Difficult to maintain and update

---

## 14. Security Vulnerability - Missing Authorization

**Problem:** Inadequate access control
```ruby
class DocumentsController < ApplicationController
  before_action :authenticate_user!

  def show
    @document = Document.find(params[:id])
    send_file @document.file_path
  end

  def destroy
    @document = Document.find(params[:id])
    @document.destroy
    redirect_to documents_path
  end

  def update
    @document = Document.find(params[:id])
    @document.update(document_params)
    redirect_to @document
  end

  private

  def document_params
    params.require(:document).permit(:title, :content, :user_id)
  end
end
```

**Issues to Identify:**
- No authorization checks (can access any document)
- Allowing user_id modification in params
- No file path validation for send_file
- Missing access control for different actions

---

## 15. Performance Issue - Inefficient Background Processing

**Problem:** Poor job design and error handling
```ruby
class EmailDigestWorker
  include Sidekiq::Worker

  def perform
    User.all.each do |user|
      posts = Post.where(created_at: 1.week.ago..Time.now)
      
      if posts.any?
        digest_content = ""
        posts.each do |post|
          digest_content += "#{post.title}\n#{post.summary}\n\n"
        end
        
        UserMailer.weekly_digest(user.email, digest_content).deliver_now
      end
    end
  end
end
```

**Issues to Identify:**
- Processing all users in single job
- N+1 queries for posts
- No error handling or retry logic
- Building large strings in memory
- Synchronous email delivery in background job
- No job batching or pagination
