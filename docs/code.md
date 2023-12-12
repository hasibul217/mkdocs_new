# **Architecture**
###MVT 
Our project leverages the Model-View-Template (MVT) architecture, a proven design pattern for building web applications. This approach promotes separation of concerns and leads to cleaner, more maintainable code. Here's a brief overview of each component:

**Model:**

* Represents the data structure and business logic of the application.
* Defines the database schema through models, which are Python classes.
* Interacts with the database to perform CRUD (Create, Read, Update, Delete) operations.

**View:**

* Handles the user interface and user input processing.
* Retrieves data from the Model and passes it to the Template.
* Contains the application's business logic related to user interactions.

**Template:**

* Defines the presentation layer and the structure of the output.
* Uses Django's template language to dynamically generate HTML.
* Receives data from the View and renders it for display to the user.


<br>

## **Some main functionalities of our project**
### **User account management**
Confirm registration by emeil verification.
```python
def confirm_registration(request,uidb64,token):
    try:
        uid = force_str(urlsafe_base64_decode(uidb64))
        user = User.objects.get(pk=uid)
    except (TypeError,ValueError,OverflowError,User.DoesNotExist):
        user = None

    if user is not None and generate_token.check_token(user,token):
        user.is_active = True
        user.save()
        login(request,user)
        messages.error(request, "Your Account has been activated!!")
        return redirect('login')
    else:
        return render(request,'activation_failed.html')
```
Make sure user is logged in
```python
def user_exists(view_func):
    def _wrapped_view(request, *args, **kwargs):
        username = request.user.username
        try:
            user = NewUser.objects.get(username=username)
        except NewUser.DoesNotExist:
            user = None
            messages.error(request, "You have been logged out successfully!")
            return redirect('welcoming_page')
        return view_func(request, user=user, *args, **kwargs)

    return _wrapped_view
```
### **Posting**
**Post for loan**

Model
```python
class LoanRequest(models.Model):
    loan_postid = models.AutoField(primary_key=True)
    created_at = models.DateTimeField(auto_now_add=True)
    loan_post = models.CharField(max_length=500)
    loan_amount = models.FloatField(default=0)
    loan_postimage = models.ImageField(upload_to='Files/LoanPost/')
    isactive = models.BooleanField(default=True)
    userid = models.ForeignKey(NewUser, on_delete=models.CASCADE)
    transaction_happen = models.BooleanField(default=False)

    def __str__(self):
        return f"LoanPost {self.loan_postid} by {self.userid}"

```
View
```python
def create_loan_post(request, user):
    if request.method == 'POST':
        post_content = request.POST.get('post')
        amount = request.POST.get('amount')
        post_image = request.FILES.get('post_image')

        if post_content and amount:
            loan_post = LoanRequest.objects.create(
                loan_post=post_content,
                loan_amount=amount,
                loan_postimage=post_image,
                userid=user
            )
            loan_post.save()
            messages.error(request, "Loan post created successfully!")
            return redirect('core_home')
        else:
            messages.error(request, "Error creating loan post. Please fill in all required fields.")
            return redirect('core_home')

    return redirect('core_home')
```
**Post for donation**

Model
```python
class DonationRequest(models.Model):
    donation_postid = models.AutoField(primary_key=True)
    created_at = models.DateTimeField(auto_now_add=True)
    donation_post = models.CharField(max_length=500)
    donation_amount = models.FloatField(default=0)
    donation_postimage = models.ImageField(upload_to='Files/DonationPost/')
    isactive = models.BooleanField(default=True)
    userid = models.ForeignKey(NewUser, on_delete=models.CASCADE)
    transaction_happen = models.BooleanField(default=False)

    def __str__(self):
        return f"DonationPost {self.donation_postid} by {self.userid}"
```
View
```python
def create_donation_post(request, user):
    if request.method == 'POST':
        post_content = request.POST.get('post')
        amount = request.POST.get('amount')
        post_image = request.FILES.get('post_image')
        if post_content and amount:
            donation_post = DonationRequest.objects.create(
                donation_post=post_content,
                donation_amount=amount,
                donation_postimage=post_image,
                userid=user
            )
            donation_post.save()
            messages.error(request, "Donation post created successfully!")
            return redirect('core_home')
        else:
            messages.error(request, "Error creating donation post. Please fill in all required fields.")
            return redirect('core_home')
    else:
        return redirect('core_home')
```
### **Add Balance**

Model
```python
class AddBalance(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)
    account_id = models.AutoField(primary_key=True)
    user = models.ForeignKey(NewUser, on_delete=models.CASCADE)
    available_balance = models.FloatField(default=0.00)
    is_first_transaction = models.BooleanField(default=True)

    def __str__(self):
        return f'Balance {self.available_balance} of {self.user.userid}'
```

View
```python
def core_statistics(request, user):
    if request.method == 'POST':
        phone = float(request.POST.get('phone'))
        amount = float(request.POST.get('amount', 0))
        add_balance, created = AddBalance.objects.get_or_create(user=user, is_first_transaction=True)

        if not created:
            add_balance.available_balance += amount
            add_balance.is_first_transaction = False
        else:
            add_balance.available_balance = amount

        add_balance.save()
        messages.error(request, f"Balance Added Successfully From the Number {phone} !!")
    available_balance = AddBalance.objects.all()
    return render(request, 'core/statistics.html', {'user': user,'available_balance':available_balance})
```



### **Donation**

Model
```python
class AddDonation(models.Model):
    donation_id = models.AutoField(primary_key=True)
    donated_to = models.ForeignKey(NewUser, on_delete=models.CASCADE, related_name='donations_received')
    donation_post = models.ForeignKey(DonationRequest, on_delete=models.CASCADE, related_name='donations')
    donated_user = models.ForeignKey(NewUser, on_delete=models.CASCADE, related_name='donations_made')
    amount = models.FloatField(default=0.00)
    wish = models.CharField(max_length=200)

    def __str__(self) -> str:
         return f'{self.donated_user.userid} donated {self.amount}'
```

View
```python
if add_balances.exists():
            latest_add_balance = add_balances.first()

            if latest_add_balance.available_balance >= amount:

                add_donation = AddDonation(
                    donated_to=donated_to_user,
                    donation_post=donation_post,
                    donated_user=donated_user,
                    amount=amount,
                    wish=wish,
                )
                add_donation.save()

                latest_add_balance.available_balance -= amount
                latest_add_balance.save()
                messages.error(request, "Thanks for your donation, check other posts.")
```

### **Comment**
```python
class Message(models.Model):
    name = models.CharField(max_length=50)
    email = models.EmailField(max_length=100)
    comment = models.TextField(max_length=300)


    def __str__(self):
        return f"From {self.name}"
```
### **Reporting**
```python
class Report(models.Model):
    report_at = models.DateTimeField(auto_now_add=True)
    report_from = models.ForeignKey(NewUser, on_delete=models.CASCADE, related_name='reports_sent')
    report_to = models.ForeignKey(NewUser, on_delete=models.CASCADE, related_name='reports_received')
    report_reason = models.CharField(max_length=50)
    short_description = models.CharField(max_length=200)

    def __str__(self):
        return f"{self.report_from.userid} reported {self.report_to.userid}"
```