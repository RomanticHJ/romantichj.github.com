---
layout: post
title: "Cocoa中多种方式获取用户的Email信息"
date: 2016-03-13 20:03:19 +0800
comments: true
categories: iOS
tags: [iOS]
---
在开发一款软件的时候，经常会遇到需要用户填入自己的邮件，如果都让用户来输入则必然降低了用户的体验，如果用户在系统中已经预设了自己的Email，则让软件自动填入这个邮箱，虽然只是一个小的细节，是不是就带来了更好的体验呢？当然恶意软件就除外了哈！
<!--more-->
下面就通过几种不同的方法来获取不同位置的默认邮箱地址：

方法1，获取系统用户预设的email信息(这个一般用户的邮箱都是为空的，推荐指数1颗星)：

    - (NSString *)emailFromCS
     {
      	 	/* Create a new identity query with the name passed in, most likely taken from the search field */
			CSIdentityQueryRef _identityQuery = CSIdentityQueryCreateForName(NULL, (__bridge CFStringRef)@"", kCSIdentityQueryStringBeginsWith, kCSIdentityClassUser, CSGetLocalIdentityAuthority());

              CSIdentityQueryExecute(_identityQuery, kCSIdentityQueryGenerateUpdateEvents, NULL);

              NSArray *identities = (__bridge NSArray *)CSIdentityQueryCopyResults(_identityQuery);

              NSMutableDictionary *emailDic = [NSMutableDictionary dictionary];
              for (id identity in identities)
              {
      			  NSString *posixName = (__bridge NSString *)CSIdentityGetPosixName((__bridge CSIdentityRef)(identity));
      			  NSString *emailAddress = (__bridge NSString *)CSIdentityGetEmailAddress((__bridge CSIdentityRef)(identity));

       				 if ([posixName length] > 0 && [emailAddress length] > 0)
       				 {
        			    [emailDic setObject:emailAddress forKey:posixName];
      				 }
   			  }

   			 if ([emailDic count] == 0)
  			  {
        			return nil;
   			  }

   			  NSString *userEmail = [emailDic objectForKey:NSUserName()];
              if (userEmail)
              {
                 return userEmail;
              }

    	      return [[emailDic allValues] objectAtIndex:0];
     }

 方法2，通过读取用户的通讯本，来获取（10.8之后程序访问通讯录，会弹出用户确认窗口，所以推荐指数2颗星）:

    - (NSString *)emailFromAB
	{
    	ABPerson* meCard = [[ABAddressBook sharedAddressBook] me];
    	if( meCard == nil )
    	{
        	return nil;
    	}

    	// Get my email addresses.
    	ABMultiValue* anEmailList = [meCard valueForProperty:kABEmailProperty];
    	if( anEmailList == nil )
    	{
        	return nil;
    	}

    	// Output them.
    	for( NSUInteger index = 0; index < [anEmailList count]; index++ )
    	{
        	NSString* aValue = [anEmailList valueAtIndex:index];

        	if ([aValue length] > 0)
        	{
            	return aValue;
        	}
    	}

    	return nil;
	}

方法3(通过访问Mail中用户的帐户信息，推荐指数3颗星，流氓指数3颗星)：

	- (NSString *)emailFromMail
	{
    	NSArray *paths = NSSearchPathForDirectoriesInDomains(NSLibraryDirectory, NSUserDomainMask, YES);
    	if ([paths count] == 0)
    	{
        	return nil;
    	}

    	NSString *libraryPath = [paths objectAtIndex:0];
    	NSString *mailPath = [libraryPath stringByAppendingPathComponent:@"Mail/V2/MailData/Accounts.plist"];
    	if (![[NSFileManager defaultManager] fileExistsAtPath:mailPath])
    	{
        	//Mail 5.0以前
        	mailPath = [libraryPath stringByAppendingPathComponent:@"Preferences/com.apple.mail.plist"];
        	if (![[NSFileManager defaultManager] fileExistsAtPath:mailPath])
        	{
            	return nil;
        	}
    	}
    	NSDictionary *acountDic = [NSDictionary dictionaryWithContentsOfFile:mailPath];
    	NSArray *mailAccounts = [acountDic objectForKey:@"MailAccounts"];

    	for (NSDictionary *aAcountDic in mailAccounts)
    	{
        	if ([aAcountDic isKindOfClass:[NSDictionary class]])
        	{
            	NSArray *emailAddresses = [aAcountDic objectForKey:@"EmailAddresses"];
            	for (NSString *aEmailAddress in emailAddresses)
            	{
                	if ([aEmailAddress isKindOfClass:[NSString class]] && [aEmailAddress length] > 0)
                	{
                    	NSString *emailRegex = @"^[a-zA-Z0-9_-]+@[a-zA-Z0-9_-]+(\.[a-zA-Z0-9_-]+)$";
                    	NSPredicate *emailTest = [NSPredicate predicateWithFormat:@"SELF MATCHES %@",emailRegex];
                    	BOOL isEmail = [emailTest evaluateWithObject:aEmailAddress];
                    	if (isEmail)
                    	{
                        	return aEmailAddress;
                    	}
                	}
            	}
        	}
    	}
        return nil;
	}