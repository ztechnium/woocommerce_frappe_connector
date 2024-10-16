from __future__ import unicode_literals
import frappe
from frappe import _
import requests.exceptions
from .woocommerce_requests import get_woocommerce_customers, post_request, put_request
from .utils import make_woocommerce_log
import re

def sync_customers():
    woocommerce_customer_list = []
    sync_woocommerce_customers(woocommerce_customer_list)
    frappe.local.form_dict.count_dict["customers"] = len(woocommerce_customer_list)

def sync_woocommerce_customers(woocommerce_customer_list):
    for woocommerce_customer in get_woocommerce_customers():
        # import new customer or update existing customer
        if not frappe.db.get_value("Customer", {"woocommerce_customer_id": woocommerce_customer.get('id')}, "name"):
            #only synch customers with address
            if woocommerce_customer.get("billing").get("address_1") != "" and woocommerce_customer.get("shipping").get("address_1") != "":
                create_customer(woocommerce_customer, woocommerce_customer_list)
        else:
            update_customer(woocommerce_customer)

def update_customer(woocommerce_customer):
    return

def create_customer(woocommerce_customer, woocommerce_customer_list):
    import frappe.utils.nestedset

    woocommerce_settings = frappe.get_doc("WooCommerce Config", "WooCommerce Config")
    
    cust_name = (woocommerce_customer.get("first_name") + " " + (woocommerce_customer.get("last_name") \
        and  woocommerce_customer.get("last_name") or "")) if woocommerce_customer.get("first_name")\
        else woocommerce_customer.get("email")
        
    try:
        # try to match territory
        country_name = get_country_name(woocommerce_customer["billing"]["country"])
        if frappe.db.exists("Territory", country_name):
            territory = country_name
        else:
            territory = frappe.utils.nestedset.get_root_of("Territory")
        customer = frappe.get_doc({
            "doctype": "Customer",
            "name": woocommerce_customer.get("id"),
            "customer_name" : cust_name,
            "woocommerce_customer_id": woocommerce_customer.get("id"),
            "sync_with_woocommerce": 0,
            "customer_group": woocommerce_settings.customer_group,
            "territory": territory,
            "customer_type": _("Individual")
        })
        customer.flags.ignore_mandatory = True
        customer.insert()
        
        if customer:
            create_customer_address(customer, woocommerce_customer)
            create_customer_contact(customer, woocommerce_customer)
    
        woocommerce_customer_list.append(woocommerce_customer.get("id"))
        frappe.db.commit()
        make_woocommerce_log(title="create customer", status="Success", method="create_customer",
            message= "create customer",request_data=woocommerce_customer, exception=False)
            
    except Exception as e:
        if e.args[0] and e.args[0].startswith("402"):
            raise e
        else:
            make_woocommerce_log(title=e, status="Error", method="create_customer", message=frappe.get_traceback(),
                request_data=woocommerce_customer, exception=True)
        
def create_customer_address(customer, woocommerce_customer):
    billing_address = woocommerce_customer.get("billing")
    shipping_address = woocommerce_customer.get("shipping")
    
    if billing_address:
        country = get_country_name(billing_address.get("country"))
        if not frappe.db.exists("Country", country):
            country = "Switzerland"
        try :
            frappe.get_doc({
                "doctype": "Address",
                "woocommerce_address_id": "Billing",
                "woocommerce_company_name": billing_address.get("company") or '',
                "address_title": customer.name,
                "address_type": "Billing",
                "address_line1": billing_address.get("address_1") or "Address 1",
                "address_line2": billing_address.get("address_2"),
                "city": billing_address.get("city") or "City",
                "state": billing_address.get("state"),
                "pincode": billing_address.get("postcode"),
                "country": country,
                "phone": billing_address.get("phone"),
                "email_id": billing_address.get("email"),
                "links": [{
                    "link_doctype": "Customer",
                    "link_name": customer.name
                }],
                "woocommerce_first_name": billing_address.get("first_name"),
                "woocommerce_last_name": billing_address.get("last_name")
            }).insert()

        except Exception as e:
            make_woocommerce_log(title=e, status="Error", method="create_customer_address", message=frappe.get_traceback(),
                    request_data=woocommerce_customer, exception=True)

    if shipping_address:
        country = get_country_name(shipping_address.get("country"))
        if not frappe.db.exists("Country", country):
            country = "Switzerland"
        try :
            frappe.get_doc({
                "doctype": "Address",
                "woocommerce_address_id": "Shipping",
                "woocommerce_company_name": shipping_address.get("company") or '',
                "address_title": customer.name,
                "address_type": "Shipping",
                "address_line1": shipping_address.get("address_1") or "Address 1",
                "address_line2": shipping_address.get("address_2"),
                "city": shipping_address.get("city") or "City",
                "state": shipping_address.get("state"),
                "pincode": shipping_address.get("postcode"),
                "country": country,
                "phone": shipping_address.get("phone"),
                "email_id": shipping_address.get("email"),
                "links": [{
                    "link_doctype": "Customer",
                    "link_name": customer.name
                }],
                "woocommerce_first_name": shipping_address.get("first_name"),
                "woocommerce_last_name": shipping_address.get("last_name")
            }).insert()
            
        except Exception as e:
            make_woocommerce_log(title=e, status="Error", method="create_customer_address", message=frappe.get_traceback(),
                request_data=woocommerce_customer, exception=True)

def sanitize_phone_number(phone):
    # Convert Arabic/Indian numerals to Western numerals
    arabic_to_western = str.maketrans("٠١٢٣٤٥٦٧٨٩", "0123456789")
    sanitized_phone = phone.translate(arabic_to_western)
    
    # Remove any non-numeric characters
    sanitized_phone = re.sub(r'\D', '', sanitized_phone)

    # You can add additional checks if needed, like ensuring it has a valid length
    if len(sanitized_phone) < 8 or len(sanitized_phone) > 15:  # Example condition for phone number length
        return None

    return sanitized_phone

def create_customer_contact(customer, woocommerce_customer):
    try:
        billing_info = woocommerce_customer.get("billing")
        shipping_info = woocommerce_customer.get("shipping")
        
        # Dynamically process the phone numbers from both billing and shipping
        phone_numbers = {
            "billing": billing_info.get("phone"),
            "shipping": shipping_info.get("phone") if shipping_info else None
        }

        first_name = billing_info.get("first_name")
        last_name = billing_info.get("last_name")
        email = billing_info.get("email")

        new_contact = frappe.get_doc({
            "doctype": "Contact",
            "first_name": first_name,
            "last_name": last_name,
            "links": [{
                "link_doctype": "Customer",
                "link_name": customer.name
            }]
        })
        
        # Add the email if available
        if email:
            new_contact.append("email_ids", {
                "email_id": email,
                "is_primary": 1
            })

        # Loop through the phone numbers and sanitize them
        for label, phone in phone_numbers.items():
            if phone:
                sanitized_phone = sanitize_phone_number(phone)
                if sanitized_phone:
                    # Append the sanitized phone to the contact
                    new_contact.append("phone_nos", {
                        "phone": sanitized_phone,
                        "is_primary_phone": 1 if label == "billing" else 0,  # Make billing primary, shipping secondary
                        "phone_type": label.capitalize()  # Label the phone type (Billing/Shipping)
                    })
        
        new_contact.insert()

    except frappe.exceptions.InvalidPhoneNumberError as e:
        make_woocommerce_log(title="Invalid Phone Number", status="Error", method="create_customer_contact",
                             message=f"Invalid phone number in {label}: {phone}", request_data=woocommerce_customer, exception=True)
    except Exception as e:
        make_woocommerce_log(title=e, status="Error", method="create_customer_contact",
                             message=frappe.get_traceback(), request_data=woocommerce_customer, exception=True)

def get_country_name(code):
    country_name = ''
    country_names = """SELECT `country_name` FROM `tabCountry` WHERE `code` = '{0}'""".format(code.lower())
    for _country_name in frappe.db.sql(country_names, as_dict=1):
        country_name = _country_name.country_name
    return country_name
