#cAPIC WORKSHOP

### Go to the terminal and run: 

	docker run -it devarshishah3/capicwkshp:latest /bin/bash
	

### What does this docker container have: 
1.	Ubuntu distribution of Linux with Terraform binary installed 
2.	ACI Python COBRA SDK for ACI 4.1
		
		cd /cobra
		ls
		EGG-INFO                       acicobra-4.1_0.183a-py2.7.egg.bkp  acimodel-4.1_0.183a-py2.7.egg.bkp
		acicobra-4.1_0.183a-py2.7.egg  acimodel-4.1_0.183a-py2.7.egg      cobra

<em>.egg</em> files of the aci model and the methods availble though COBRA are listed here
		
		cd /cobra/cobra
		ls
		__init__.py  __init__.pyc  internal  mit  model  modelimpl  services.py  services.pyc 

This lists the classes and the model methods.
<strong>Please refer to this while using COBRA SDK</strong>



###	Credentials
	APIC: https://13.56.122.16

	username: lab-user-{usernumber}

	password: CiscoLive2019

replace {usernumber} with the number assigned to your workstation
eg: user1 will be "lab-user-1", user2 will be "lab-user-2"  and so on

	cd capic-wkshp/


# LAB
- 1. Create a session to the APIC 
- 2. Tenant
	-	3. Create a VRF
	-  4.	Create a Cloud Context Domain 
		-  5. Create a Relation to the VRF
		-  6. Create a Relation to an AWS region
		-  7. Create a CIDR
			-  	8. Create a Subnet and attch it to a Availability Zone
	-  9. Create Filters and Filter Entries
	- 	10. Create a Contract, Contract Subject and form a relationship to the filters  
	-  11.	Create an Cloud App
		-  		12. Create an external EPG.
		-  	13.	 Create cloud 2 EPGs
	     	  
### Variables


All variables are defined in <em>variables.py</em>

<strong>Solution is available in solution.py and variables.py.bkp

But you are encouraged to try it on your own. Copy-Paste is absoultely ACCEPTABLE</strong>

Use Your favourite editor to create a file workshop.py

###Task 0:
Import the libraires and define the main method 

	#!/usr/bin/env python
	import requests
	import cobra.mit.access
	import cobra.mit.session
	import cobra.mit.request
	from credentials import *
	from variables import *
	import json

	def main():
	 	"""
    	This function creates the new Tenant with a VRF, CloudCtx Profile, COntracts and Cloud APp.
    	"""
		print ("Main method")
	
	if __name__ == '__main__':
    	main()
    	
On the console:

	python workshop.py

Make sure there are no errors

Continue editing workshop.py

###Task 1:
#### Create a session to the APIC 	
In main method:

	def main():
    """
    This function creates the new Tenant with a VRF, CloudCtx Profile, COntracts and Cloud APp.
    """
    # create a session and define the root
    	requests.packages.urllib3.disable_warnings()
    	auth = cobra.mit.session.LoginSession(URL, LOGIN, PASSWORD)
    	session = cobra.mit.access.MoDirectory(auth)
    	session.login()
On the console:

	python workshop.py

Make sure there are no errors

###Task 2:
####	Create a Tenant
A Tenant is a container for all network, security, troubleshooting and L4 – 7 service policies.   Tenant resources are isolated from each other, allowing management by different administrators. 


In this task, you will first ckeck if the Tenant exists

Edit <em>variables.py</em>

	# Tenant Name
	TENANT = "cl-workshop"
	
Continue editing <em>workshop.py</em>

	def test_tenant(tenant_name, apic_session):
    	"""
    	This function tests if the desired Tenant name is already in use.
    	If the name is already in use, it will exit the script early.

    	:param tenant_name: The new Tenant's name
    	:param apic_session: An established session with the APIC
    	"""
    	# build query for existing tenants
    	tenant_query = cobra.mit.request.ClassQuery('fvTenant')
    	tenant_query.propFilter = 'eq(fvTenant.name, "{}")'.format(tenant_name)

    	# test for truthiness
    	if apic_session.query(tenant_query):
        	print("\nTenant {} has already been created on the APIC\n".format(tenant_name))


	def main():
    	"""
    	This function creates the new Tenant with a VRF, CloudCtx Profile, COntracts and Cloud APp.
    	"""
    	# create a session and define the root
    	requests.packages.urllib3.disable_warnings()
    	auth = cobra.mit.session.LoginSession(URL, LOGIN, PASSWORD)
    	session = cobra.mit.access.MoDirectory(auth)
    	session.login()

    	root = cobra.model.pol.Uni('')

    	# test if tenant name is already in use
    	test_tenant(TENANT, session)

On the console:

	python workshop.py

Make sure there are no errors

You will see an output that this tenant has already been created 

Next you will create a Tenant managed object (MO) and commit it to APIC

<em>This is to test the idempotency of APIC</em>


Continue editing <em>workshop.py</em>
	
	def main():
    	"""
    	This function creates the new Tenant with a VRF, CloudCtx Profile, COntracts and Cloud APp.
    	"""
    	# create a session and define the root
    	requests.packages.urllib3.disable_warnings()
    	auth = cobra.mit.session.LoginSession(URL, LOGIN, PASSWORD)
    	session = cobra.mit.access.MoDirectory(auth)
    	session.login()

    	root = cobra.model.pol.Uni('')

    	# test if tenant name is already in use
    	test_tenant(TENANT, session)
		# model new tenant configuration
    	# each tenant in cAPIC is maps to an AWS user account
    	tenant = cobra.model.fv.Tenant(root, name=TENANT)

    	#submit the configuration to the apic and print a success message
    	config_request = cobra.mit.request.ConfigRequest()

    	config_request.addMo(tenant)

    	session.commit(config_request)

    	config_data = json.loads(config_request.data)

    	print("\nNew Objects created:\n\n{}\n".format(json.dumps(config_data, indent=4, sort_keys=True)))


On the console:

	python workshop.py
	
You should see a json output with the tenant name and the parameters you put in


	
Now you have a hang of it. Let's march on

###Task 3:
#### Create a VRF
Private networks (also called VRFs or contexts) are defined within a tenant to allow isolated and potentially overlapping IP address space

VRF is a child object of a tenant. 

Edit <em>variables.py</em>
	
	#VRF Name
	VRF = "user-vrf-{usernumber}" #input your user number

Continue editing <em>workshop.tf</em>
Define a method <em>create_vrf</em> and call it in main after the tenant object has been created
	
	def create_vrf(tenant):    
    	# create a VRF 
    	vrf = cobra.model.fv.Ctx(tenant, name=VRF)
    
	def main():
	
		#
		#  Your previous code here
		#
	
		# Creating a VRF give you the ability to have unique forwarding and application policy domain 
    	# and have overlapping overlapping IP address in the same tenant
    	# In case of cAPIC, VRF maps to an AWS VPC
    	# VRF is a child of a Tenant
    	create_vrf(tenant)
	
On the console:

	python workshop.py
	
You should see a json output with the tenant, vrf and other objects.


	
###	Task 4:
#### Create a Cloud Context Profile:
Cloud Context Profile is a new class in ACI whihc ties the AWS Regions, CIDRs and Subnets to a VRF

Cloud Context Profile is a child of a Tenant

Cloud Context Profile needs to be associated to a VRF


Edit <em>variables.py</em>
	
	CLOUDCTXPROFILE = "cloudcontext-{usernumber}" #input your usernumber
	REGIONPROFILE = "uni/clouddomp/provp-aws/region-us-west-1"
	CIDRADDR = "192.168.0.0/16" #PLEASE DO NOT CHANGE
	SUBNETIP = "192.168.11.0/24"  #192.168.1<usernumber>.0/24  ex:user1:92.168.11.0/24, user2:192.168.12.0/24, user3:192.168.13.0/24
	ZONEPROFILE = "uni/clouddomp/provp-aws/region-us-west-1/zone-us-west-1a"

Continue editing <em>workshop.py</em>

Define a method create_cloud_ctx_profile(tenant)and call it in main after the tenant object has been created

	def create_cloud_ctx_profile(tenant):

    	#create a Cloud Context Profile for the tenant
    	cloud_ctx_profile = cobra.model.cloud.CtxProfile(tenant, name=CLOUDCTXPROFILE)


	def main():
	
		#
		#  Your previous code here
		#
	
	    # Cloud Context Profile is a new class in ACI which ties the AWS Regions, CIDRs and Subnets to a VRF
    	# Cloud Context Profile is a child of a Tenant
    	# Cloud Context Profile needs to be associated to a VRF 
    	create_cloud_ctx_profile(tenant)

On the console:

	python workshop.py
	
You should see a json output with the tenant, vrf , cloud context profile and other objects.


### Task 5. 
####Create a Relation to the VRF
In the method create_cloud_context_profile, associate cloud context to the VRF

Continue editing <em>workshop.py</em>

	#attach Cloud Context Profile to a VRF
    	attach_cloud_vrf = cobra.model.cloud.RsToCtx(cloud_ctx_profile, tnFvCtxName=VRF)

On the console:

	python workshop.py
	
You should see a json output with the tenant, vrf , cloud context profile and other objects.

###Task 6. 
####Create a Relation to an AWS region

In the method create_cloud_context_profile, attach context profile to an AWS region

Continue editing <em>workshop.py</em>

		#attach Cloud Context Profile to a Cloud Region
    	attach_cloud_region = cobra.model.cloud.RsCtxProfileToRegion(cloud_ctx_profile, tDn=REGIONPROFILE)

On the console:

	python workshop.py
	
You should see a json output with the tenant, vrf , cloud context profile and other objects.



###Task 7: 
####Create a CIDR

In the method create_cloud_context_profile, create a child object of the Cloud Context Profile, Cloud CIDR.

Continue editing <em>workshop.py</em>

	#create Cloud CIDR
    	cloud_cidr = cobra.model.cloud.Cidr(cloud_ctx_profile, addr=CIDRADDR, primary="yes")

On the console:

	python workshop.py
	
You should see a json output with the tenant, vrf , cloud context profile and other objects.


###Task 8: 
####Create a Subnet and Attach it to AWS Cloud Zone

In the method create_cloud_context_profile, create a child object of the previously created cloud CIDR, Cloud Subnet. Attach that Subnet to the AWS region

Continue editing <em>workshop.py</em>

		#create Cloud Subnet
    	cloud_subnet = cobra.model.cloud.Subnet(cloud_cidr, ip=SUBNETIP)

		#attach subnet to cloud zone
    	attach_cloud_zone = cobra.model.cloud.RsZoneAttach(cloud_subnet, tDn=ZONEPROFILE)

On the console:

	python workshop.py
	
You should see a json output with the tenant, vrf , cloud context profile and other objects.




	
###Task 9:
#### Create a Filter and Filter Entry
A filter classifies a collection of network packet  attributes

Filter is a child object of a tenant. You will have to pass the tenant's (parents) DN (Distinguished Name)
Filter entry is a network packet  attributes. It selects a packets based on the attributes specified.

Create a filter entry for ssh and icmp

Filter Entry is a child object of a Filter . You will have to pass the Filter's (parents) DN (Distinguished Name)

Create 2 filters and filter entries. One related to ICMP and the second related to SSH

Edit <em>variables.py</em>
	
	#Filter information
	ICMPFILTER = "icmpfilter-{usernumber}" #input usernumber
	ICMP = "icmpentry-{usernumber}" #input usernumber  
	SSHFILTER = "sshfilter-{usernumber}" #input usernumber
	SSH = "sshentry-{usernumber}" #input usernumber

Continue editing <em>workshop.py</em>

Define a method <em>create_filters</em> and call it in main after the tenant object has been created

	def create_filters(tenant):
		# create a filter for ICMP
    	# create a filter for ICMP 
    	icmp_filter = cobra.model.vz.Filter(tenant, name=ICMPFILTER)

    	# create an entry for ICMP and associate it to the ICMP filter
    	icmp_entry = cobra.model.vz.Entry(icmp_filter, name=ICMP,
                                                    etherT="ip",
                                                    prot = "icmp",
                                                    stateful = "yes")


    	# create a filter for SSH
    	ssh_filter = cobra.model.vz.Filter(tenant, name=SSHFILTER)
    	ssh_entry = cobra.model.vz.Entry(ssh_filter, name=SSH,
                                        etherT="ip",
                                        prot="tcp",
                                        dToPort=22,
                                        dFromPort=22)

	def main():
	
		#
		#  Your previous code here
		#
		# Filters allow you to Specify the ethertype, protocol, source and dest ports, etc
    	# Filter is a child of a Tenant
    	# Filter has entries
    	# Filter is equivalent to Security Group Rule
    	create_filters(tenant)




On the console:

	python workshop.py
	
You should see a json output with the tenant, vrf , cloud context profile, filter , filter entry and other objects.
	



###Task 10:
#### Create a Contact, Contract Subject and Attach it to the filter 
Contract is a set of rules governing communication between EndPoint Groups

Contract is a child object of a tenant. You will have to pass the tenant's (parents) DN (Distinguished Name)

Contract Subject sets the Permit/Deny, Qos policies of the Contract. Relationship needs to be created from the subject contract to filter entry.

Contract Subject is a child object of a Contract. You will have to pass the Contract's (parents) DN (Distinguished Name)

Edit <em>variables.py</em>
	
	CONTRACT = "accesscontract-{usernumber}" #input usernumber
	SUBJECT = "accesssubject-{usernumber}" #input usernumber
	
Continue editing <em>workshop.py</em>

Define a method <em>create_contract</em> and call it in main after the tenant object has been created

	def create_contract(tenant):

    	# create a contract for the tenant
    	contract = cobra.model.vz.BrCP(tenant, name=CONTRACT)

    	#create a subject for the contract
    	subject = cobra.model.vz.Subj(contract, name=SUBJECT)

    	#attach the ICMP filter to the subject
    	attach_icmp = cobra.model.vz.RsSubjFiltAtt(subject, tnVzFilterName=ICMPFILTER)

    	#attach the ICMP filter to the subject
    	attach_ssh = cobra.model.vz.RsSubjFiltAtt(subject, tnVzFilterName=SSHFILTER)


	def main():
	
		#
		#  Your previous code here
		#
		# Contarct Binds multiple filters together
    	# Contract is equivalent to Security Group Rule
    	create_contract(tenant)

On the console:

	python workshop.py
	
You should see a json output with the tenant, vrf , cloud context profile, contract, contract subject and other objects.	

	
###Task 11:
#### Create a Cloud App

Cloud Application is a collection of end points groups and contract between them

Cloud Application is a child object of a tenant. You will have to pass the tenant's (parents) DN (Distinguished Name)

Edit <em>variables.py</em>

	#Cloud Application Name
	CLOUDAPPLICATION = "user-ap-{usernumber}" #input usernumber
	
Continue editing <em>workshop.py</em>

Define a method <em>create_cloud_app</em> and call it in main after the tenant object has been created. Return the cloud_app object from the method

	def create_cloud_app(tenant):

    	# create a cloudApp. This is similar to Application Profile on On-prem APIC
    	cloud_app = cobra.model.cloud.App(tenant, name=CLOUDAPPLICATION)

    	return cloud_app


	def main():
	
		#
		#  Your previous code here
		#
		# Cloud App is equivalent to an Application Profile(on-prem APIC)
    	# Cloud App is a child of a Tenant
    	# It is a logical container for all policies related to one application
    	cloud_app = create_cloud_app(tenant)


On the console:

	python workshop.py
	
You should see a json output with the tenant, vrf , cloud context profile, filter , filter entry and other objects.

###Task 12:
#### Create cloud external EPG. 	Attach EPG to a Cloud Context Profile and a Contract.


End Points are devices which attach to the network 

e.g:
EC2 instances
S3 bucket 


EPG is a logical collection of End Points 

An EPG could to be linked to a Cloud Context Profile 


A consumed contract (outbound rule) and a provided contract (inbound rule) must also be added for inter EPG communication 


EPG is a child object of an Cloud App. You will have to pass the Cloud Apps's (parents) DN (Distinguished Name)


Cloud External EPG is an EPG for all internet/multiste endpoints

Cloud External EPG is a child object of a Cloud App. You will have to pass the cloud app's (parents) DN (Distinguished Name)

Continue editing <em>workshop.py</em>
Define a method <em>create_cloud_ext_epg</em> and call it in main after the cloud app object has been created. 

	def create_cloud_ext_epg(cloud_app):

    	#create a cloud external EPG for traffic coming from the internet
    	cloud_external_epg = cobra.model.cloud.ExtEPg(cloud_app, name=CLOUDEXTEPG, routeReachability ="internet")

    	#create a external EP selector for Subnets
    	cloud_ext_ep_slector = cobra.model.cloud.ExtEPSelector(cloud_external_epg,name=EXTEPSELECTOR, subnet ="0.0.0.0/0")

    	#attach the external EPG to a VRF
    	attach_ext_ep_to_vrf = cobra.model.cloud.RsCloudEPgCtx(cloud_external_epg, tnFvCtxName=VRF)

    	#create a Consumer contract for External EPG
    	consumed_contract_ext_epg = cobra.model.fv.RsCons(cloud_external_epg, tnVzBrCPName=CONTRACT)
    	#create a Provided contract for External EPG
    	provided_contract_ext_epg = cobra.model.fv.RsProv(cloud_external_epg, tnVzBrCPName=CONTRACT)

	def main():
	
		#
		#  Your previous code here
		#
		# Cloud External EPG is a  EPG for Internet or inter-site
    	# Cloud EPG is a child of the Cloud App
    	create_cloud_ext_epg(cloud_app)
    	
On the console:

	python workshop.py
	
You should see a json output with the tenant, vrf , cloud context profile, filter , filter entry and other objects.


###Task 12:
#### Create 2 Cloud EPGs. 	Attach EPG to a Cloud Context Profile and a Contract.


End Points are devices which attach to the network 

e.g:
EC2 instances
S3 bucket 


EPG is a logical collection of End Points 

An EPG could to be linked to a Cloud Context Profile 


A consumed contract (outbound rule) and a provided contract (inbound rule) must also be added for inter EPG communication 


EPG is a child object of an Cloud App. You will have to pass the Cloud Apps's (parents) DN (Distinguished Name)


Cloud EPG is an EPG for all Cloud Endpoints

Cloud EPG is a child object of a Cloud App. You will have to pass the cloud app's (parents) DN (Distinguished Name)

Edit <em>variables.py</em>

	#EPG information including EPG selector name
	EPG1 = "epg1-{usernumber}"   #input usernumber
	EPG1SELECTOR1 = "epg1select1-{usernumber}"   #input usernumber
	EPG1SELECTOR2 = "epg1select2-{usernumber}"   #input usernumber

	EPG2 = "epg2-{usernumber}"   #input usernumber
	EPG2SELECTOR1 = "epg2select1-{usernumber}"   #input usernumber
	EPG2SELECTOR2 = "epg2select2-{usernumber}"   #input usernumber



Continue editing <em>workshop.py</em>
Define a method <em>create_cloud_epg_1</em>,<em>create_cloud_epg_2</em> and call it in main after the cloud app object has been created. 

	def create_cloud_epg_1(cloud_app):
    	# create cloud EPGs 
    	cloud_epg_1 = cobra.model.cloud.EPg(cloud_app, name=EPG1)
    	# attach epg to a vrf
    	epg1_attach_vrf = cobra.model.cloud.RsCloudEPgCtx(cloud_epg_1, tnFvCtxName=VRF)
    	# create cloud EP selector
    	epg1_selector_1 = cobra.model.cloud.EPSelector(cloud_epg_1, name= EPG1SELECTOR1,
                                                        matchExpression = "custom:tag=='user1epg1'")#user<usernumber>epg1

    	#create a Consumer contract for EPG1
    	consumed_contract_epg_1 = cobra.model.fv.RsCons(cloud_epg_1, tnVzBrCPName=CONTRACT)

    	#create a Provided contract for EPG1
    	provided_contract_epg_1 = cobra.model.fv.RsProv(cloud_epg_1, tnVzBrCPName=CONTRACT)


	def create_cloud_epg_2(cloud_app):

    	# create cloud EPGs
    	cloud_epg_2 = cobra.model.cloud.EPg(cloud_app, name=EPG2)
    	# attach epg to a vrf
    	epg2_attach_vrf = cobra.model.cloud.RsCloudEPgCtx(cloud_epg_2, tnFvCtxName=VRF)
    	# create cloud EP selector
    	epg2_selector_1 = cobra.model.cloud.EPSelector(cloud_epg_2, name= EPG2SELECTOR1,
                                                        matchExpression = "custom:tag=='user1epg2'")#user<usernumber>epg2

    	#create a Consumer contract for EPG2
    	consumed_contract_epg_2 = cobra.model.fv.RsCons(cloud_epg_2, tnVzBrCPName=CONTRACT)
    	#create a Provided contract for EPG2
    	provided_contract_epg_2 = cobra.model.fv.RsProv(cloud_epg_2, tnVzBrCPName=CONTRACT)

		def create_cloud_epg_1(cloud_app):
    	# create cloud EPGs 
    	cloud_epg_1 = cobra.model.cloud.EPg(cloud_app, name=EPG1)
    	# attach epg to a vrf
    	epg1_attach_vrf = cobra.model.cloud.RsCloudEPgCtx(cloud_epg_1, tnFvCtxName=VRF)
    	# create cloud EP selector
    	epg1_selector_1 = cobra.model.cloud.EPSelector(cloud_epg_1, name= EPG1SELECTOR1,
                                                        matchExpression = "custom:tag=='user1epg1'")#user<usernumber>epg1

    	#create a Consumer contract for EPG1
    	consumed_contract_epg_1 = cobra.model.fv.RsCons(cloud_epg_1, tnVzBrCPName=CONTRACT)

    	#create a Provided contract for EPG1
    	provided_contract_epg_1 = cobra.model.fv.RsProv(cloud_epg_1, tnVzBrCPName=CONTRACT)


	def create_cloud_epg_2(cloud_app):

    	# create cloud EPGs
    	cloud_epg_2 = cobra.model.cloud.EPg(cloud_app, name=EPG2)
    	# attach epg to a vrf
    	epg2_attach_vrf = cobra.model.cloud.RsCloudEPgCtx(cloud_epg_2, tnFvCtxName=VRF)
    	# create cloud EP selector
    	epg2_selector_1 = cobra.model.cloud.EPSelector(cloud_epg_2, name= EPG2SELECTOR1,
                                                        matchExpression = "custom:tag=='user1epg2'")#user<usernumber>epg2

    	#create a Consumer contract for EPG2
    	consumed_contract_epg_2 = cobra.model.fv.RsCons(cloud_epg_2, tnVzBrCPName=CONTRACT)
    	#create a Provided contract for EPG2
    	provided_contract_epg_2 = cobra.model.fv.RsProv(cloud_epg_2, tnVzBrCPName=CONTRACT)


	
	def main():
	
		#
		#  Your previous code here
		#
		# Cloud End Point Group is a collection of devices,services or subnets that are on AWS
    	# Cloud EPG is a child of the Cloud App
    	create_cloud_epg_1(cloud_app)

    	create_cloud_epg_2(cloud_app)
    	
On the console:

	python workshop.py
	
You should see a json output with the tenant, vrf , cloud context profile, filter , filter entry and other objects.

#Output

If all the configuration went through correctly and the varibles in variables.py are input correctly, you are ready to spin up an EC2 instance

- 1. Log into the AWS Console for the correct region
- 2. Go to EC2 and Launch EC2 instance
- 3. VPC should have been created with the correct CIDR and Subnet
- 4. Select the right Subnets and an IP from that subnet. .1-.3 are reserved.
- 5. Select a Public IP
- 6. Attach the Tags created in the EP selectors for appropriate EPGs
- 7. Leave the default Security Policies.
- 8. Use <em>capic-workshop.pem</em> as the key
- 8. Repeat the same for the other EC2 and assign it to the other EPG portgroup
- 9. Make sure the EPGs are able to ping each other 






