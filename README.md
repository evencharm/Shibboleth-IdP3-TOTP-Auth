[![Apache License](http://img.shields.io/:license-apache-blue.svg)](http://www.apache.org/licenses/LICENSE-2.0.html)
[![Build Status](https://travis-ci.org/korteke/Shibboleth-IdP3-TOTP-Auth.svg?branch=master)](https://travis-ci.org/korteke/Shibboleth-IdP3-TOTP-Auth)

# THIS IS A DEVELEOPMENT FORK

# Shibboleth-IdP3-TOTP-Auth
> Working example of the TOTP authenticator. Work in progress! Refactoring needed! Localization needed.  

Google authenticator authentication module for Shibboleth IdP v3.  
The original version used the password flow and then called the totp flow.  The updated version (this fork) relies on the MFA flow to call the password flow first.
This authn flow will take the c14n principal name and perform the token validation based on that.  This authn flow will not work by itself.


Uses External LDAP, MongoDB(EXPERIMENTAL!) or Static for seed fetching.  
The original implementation of this authn flow had a separate ldap configuration.  This implementation will assume the totp seed is stored in a database that the attribute resolver has the ability to pull from.
It will also assume the attribute is encrypted using a secret key (configured in a properties file).

Working example with Vagrant https://github.com/korteke/shibboleth-vagrant

Requirements
------------

Shibboleth IdP v3.2.x  
Java 8

Installing
----------

* Compile souce code with maven - ```mvn clean package```
* Copy and extract totpauth-parent/totpauth-impl/target/totpauth-impl-0.5.1-bin.zip

Directory structure:
<pre>
├── conf
│   └── authn
├── edit-webapp
│   └── WEB-INF
│       └── lib
├── flows
│   └── authn
│       └── Totp
└── views
</pre>

* Copy conf --> $IDP-HOME/conf  
* Copy edit-webapp  --> $IDP-HOME/edit-webapp  
* Copy flows  --> $IDP-HOME/flows  
* Copy views  --> $IDP-HOME/views  

Modify $IDP_HOME/conf/idp.properties  

idp.authn.flows = Password --> idp.authn.flows = Password|Totp

Add TOTP bean to $IDP_HOME/conf/authn/general-authn.xml, to the element:
```
 "<util:list id="shibboleth.AvailableAuthenticationFlows">"
```
  New Bean
```
        <bean id="authn/Totp" parent="shibboleth.AuthenticationFlow"
                p:passiveAuthenticationSupported="true"
                p:forcedAuthenticationSupported="true">
            <property name="supportedPrincipals">
                <util:list>
                    <bean parent="shibboleth.SAML2AuthnContextClassRef"
                        c:classRef="urn:oasis:names:tc:SAML:2.0:ac:classes:TimeSyncToken" />
                </util:list>
            </property>
        </bean>
```

### Rebuild idp.war
* run $IDP-HOME/bin/build.sh
* If you need, move that war-file to containers "webapps" directory (tomcat, jetty, etc)
* Restart container

Seed Fetching
-------------
From LDAP, MongoDB, SQL, File, REST, Dummy(static)

### From LDAP - External LDAP (IDM?)
With default settings this plugin fetches token seeds from the attribute called "carLicense" which is multivalued (user can have multiple tokens).  
You can change the source attribute by editing bean "shibboleth.authn.seedAttribute" which is defined at totp-authn-config.xml.    
    
This plugin also assumes that your users unique userID is "uid" attribute.    
This can be changed by editing bean "shibboleth.authn.userAttribute" at totp-authn-config.xml.  

* Modify LDAP properties - totp-authn-beans.xml (url, userDn, password, base)  
* Make sure that bean id "shibboleth.totp.seedfetcher" is pointing to "net.kvak.shibboleth.totpauth.authn.impl.seed.LdapSeedFetcher"  

### From MongoDB - Experimental, just testing, but it works

* Modify MongoDB properties - totp-authn-config.xml (mongoDbUrl, mongoDbName)  
* Make sure that bean id "shibboleth.totp.seedfetcher" is pointing to "net.kvak.shibboleth.totpauth.authn.impl.seed.MongoSeedFetcher"  

### From Dummy - Static code

* Make sure that bean id "shibboleth.totp.seedfetcher" is pointing to "net.kvak.shibboleth.totpauth.authn.impl.seed.DummySeedFetcher"
* Register this token to your mobile device:  
![alt tag](https://raw.githubusercontent.com/korteke/Shibboleth-IdP3-TOTP-Auth/master/totp_code_qr.png)

Adding new seed to user
----------------------

~~At the moment you need to add your token codes to the repository with external process. I will create some kind of registeration flow to the IdP.~~   
TOTP login page has a button called "Register a new token" which triggers a new flow where users can register their tokens. ATM the button is visible to all users. Next version you can choose if the users can register new tokens.  

This works at the moment only with the LDAP seedFetcher.
MongoDB registeration flow is probably coming soon.  


Requesting new Authentication Context Class with Shibboleth SP
-----------------------------------------------

(for testing purpose)
Add new Session Initiator

```
<SessionInitiator type="Chaining" Location="/totp" id="totp" entityID="https://IDP-ENTITY-ID">  
  <SessionInitiator type="SAML2" acsIndex="1" template="bindingTemplate.html" authnContextClassRef="urn:oasis:names:tc:SAML:2.0:ac:classes:TimeSyncToken"/>  
</SessionInitiator>  
```

After the new SessionInitiator is registered you can call your SP with https://YOUR_SP_ADDRESS/Shibboleth.sso/totp  
That creates SAML 2.0 Authn Request where SP wants authnContextClassRef == urn:oasis:names:tc:SAML:2.0:ac:classes:TimeSyncToken  
So when the IdP receives that request it passes the request to the authn/Totp authentication flow (this module)
