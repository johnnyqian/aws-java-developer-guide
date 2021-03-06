.. Copyright 2010-2016 Amazon.com, Inc. or its affiliates. All Rights Reserved.

   This work is licensed under a Creative Commons Attribution-NonCommercial-ShareAlike 4.0
   International License (the "License"). You may not use this file except in compliance with the
   License. A copy of the License is located at http://creativecommons.org/licenses/by-nc-sa/4.0/.

   This file is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND,
   either express or implied. See the License for the specific language governing permissions and
   limitations under the License.

##############################################################
Using IAM Roles to Grant Access to AWS Resources on Amazon EC2
##############################################################

All requests to Amazon Web Services (AWS) must be cryptographically signed using credentials issued
by AWS. You can use :emphasis:`IAM roles` to conveniently grant secure access to AWS resources from
your EC2 instances.

This topic provides information about how to use IAM roles with Java SDK applications running on
EC2. For more information about IAM instances, see :ec2-ug:`IAM Roles for Amazon EC2
<iam-roles-for-amazon-ec2>` in the |EC2-ug|.

.. contents::
    :local:
    :depth: 1

.. _default-provider-chain:

The default provider chain and EC2 instance profiles
====================================================

If your application creates an AWS client using the default constructor, then the client will search
for credentials using the :emphasis:`default credentials provider chain`, in the following order:

1.  In system environment variables: :code:`AWS_ACCESS_KEY_ID` and :code:`AWS_SECRET_ACCESS_KEY`.

2.  In the Java system properties: :code:`aws.accessKeyId` and :code:`aws.secretKey`.

3.  In the default credentials file (the location of this file varies by platform).

4.  In the :emphasis:`instance profile credentials`, which exist within the instance metadata
    associated with the IAM role for the EC2 instance.

The final step in the default provider chain is available only when running your application on an
EC2 instance, but provides the greatest ease of use and best security when working with EC2
instances. You can also pass an InstanceProfileCredentialsProvider instance directly to the client
constructor to get instance profile credentials without proceeding through the entire default
provider chain. For example:

.. code-block:: java

      AmazonS3 s3Client = new AmazonS3Client(new InstanceProfileCredentialsProvider());

When using this approach, the SDK will retrieve temporary AWS credentials that have the same
permissions as those associated with the IAM role associated with the EC2 instance in its instance
profile. Although these credentials are temporary and would eventually expire,
InstanceProfileCredentialsProvider will periodically refresh them for you so that the obtained
credentials continue to allow access to AWS.

.. important:: The automatic credentials refresh happens :emphasis:`only` when you use the default
   client constructor, which creates its own :code:`InstanceProfileCredentialsProvider` as part of
   the default provider chain, or when you pass an :code:`InstanceProfileCredentialsProvider`
   instance directly to the client constructor. If you use another method to obtain or pass instance
   profile credentials, you are responsible for checking for and refreshing expired credentials.

If the client constructor can't find credentials using the credentials provider chain, it will throw
an AmazonClientException.


.. _roles-walkthrough:

Walkthrough: Using IAM roles for EC2 instances
==============================================

The following walkthrough shows you how to retrieve an object from |S3| using an |IAM| role to
manage access.

.. contents::
   :depth: 1
   :local:

.. _java-dg-create-the-role:

Create an |IAM| Role
--------------------

Create an IAM role that grants read-only access to |S3|.

**To create the IAM role**

1.  Open the IAM console.

2.  In the navigation pane, click :guilabel:`Roles`, and then click :guilabel:`Create New Role`.

3.  Enter a name for the role, and then click :guilabel:`Next Step`. Remember this name, as you'll
    need it when you launch your EC2 instance.

4.  On the :guilabel:`Select Role Type` page, under :guilabel:`AWS Service Roles`, select
    :guilabel:`Amazon EC2`.

5.  On the :guilabel:`Set Permissions` page, under :guilabel:`Select Policy Template`, select
    :guilabel:`Amazon S3 Read Only Access`. Click :guilabel:`Next Step`.

6.  On the :guilabel:`Review` page, click :guilabel:`Create Role`.


.. _java-dg-launch-ec2-instance-with-instance-profile:

Launch an EC2 Instance and Specify Your IAM Role
------------------------------------------------

You can launch an EC2 instance with an IAM role using the |EC2| console or the |sdk-java|.

*   To launch an EC2 instance using the console, follow the directions in :ec2-ug:`Launch an EC2
    Instance <ec2-launch-instance_linux>` in the |EC2-ug|. When you reach the :guilabel:`Review
    Instance Launch` page, click :guilabel:`Edit instance details`. In :guilabel:`IAM role`, specify
    the IAM role that you created previously. Complete the procedure as directed. Notice that you'll
    need to create or use an existing security group and key pair in order to connect to the
    instance.

*   To launch an EC2 instance with an IAM role using the |sdk-java|, see :doc:`run-instance`.


.. _java-dg-remove-the-credentials:

Create your Application
-----------------------

Let's build the sample application to run on the EC2 instance. First, create a directory that you
can use to hold your tutorial files (for example, :file:`GetS3ObjectApp`).

Next, copy the |sdk-java| libraries into your newly-created directory. If you downloaded the
|sdk-java| to your :file:`~/Downloads` directory, you can copy them using the following commands:

.. code-block:: sh

    cp -r ~/Downloads/aws-java-sdk-{1.7.5}/lib . cp -r ~/Downloads/aws-java-sdk-{1.7.5}/third-party .

Open a new file, call it :file:`GetS3Ojbect.java`, and add the following code:

.. literalinclude:: snippets/GetS3ObjectApp/GetS3Object.java
    :language: java
    :lines: 14-

Open a new file, call it :file:`build.xml`, and add the following lines:

.. literalinclude:: snippets/GetS3ObjectApp/build.xml
    :language: xml
    :lines: 14-

Build and run the modified program. Note that there are no credentials are stored in the program.
Therefore, unless you have your AWS credentials specified already, the code will throw
:code:`AmazonServiceException`. For example:

.. code-block:: sh

    $ ant
    Buildfile: /path/to/my/GetS3ObjectApp/build.xml

    build:
      [javac] Compiling 1 source file to /path/to/my/GetS3ObjectApp

    run:
       [java] Downloading an object
       [java] AmazonServiceException

    BUILD SUCCESSFUL


.. _java-dg-transfer-compiled-program-to-ec2-instance:

Transfer the Compiled Program to Your EC2 Instance
--------------------------------------------------

Transfer the program to your EC2 instance using secure copy (scp), along with the |sdk-java|
libraries. The sequence of commands looks something like the following. Depending on the Linux
distribution that you used, the user name might be "ec2-user", "root", or "ubuntu". To get the
public DNS name of your instance, select it in the |EC2| console, and then look for
:guilabel:`Public DNS` in the :guilabel:`Description` tab (for example,
ec2-198-51-100-1.compute-1.amazonaws.com).

.. code-block:: sh

     scp -p -i {my-key-pair}.pem GetS3Object.class ec2-user@{public_dns}:GetS3Object.class
     scp -p -i {my-key-pair}.pem build.xml ec2-user@{public_dns}:build.xml
     scp -r -p -i {my-key-pair}.pem lib ec2-user@{public_dns}:lib
     scp -r -p -i {my-key-pair}.pem third-party ec2-user@{public_dns}:third-party

In the preceding commands, :code:`GetS3Object.class` is your compiled program, :code:`build.xml` is
the ant file used to build and run your program, and the :code:`lib` and :code:`third-party`
directories are the corresponding library folders from the |sdk-java|.

The :code:`-r` switch indicates that :literal:`scp` should do a recursive copy of all of the
contents of the library and third-party directories from the |sdk-java|.

The :code:`-p` switch indicates that :literal:`scp` should preserve the permissions of the source
files when it copies them to the destination. If you are copying the files from Windows, you might
need to fix the permissions on your instance using the following command:

.. code-block:: sh

    chmod -R u+rwx GetS3Object.class build.xml lib third-party


.. _java-dg-run-the-program:

Run the Sample Program on the EC2 Instance
------------------------------------------

To run the program, connect to your EC2 instance. For more information, see :ec2-ug:`Connect to Your
Linux Instance <AccessingInstances>` in the |EC2-ug|.

If ant is not installed on your instance, you can install it using the `yum
<http://yum.baseurl.org/>`_ installer as follows:

.. code-block:: sh

    sudo yum install ant

Run the program using :code:`ant` as follows:

.. code-block:: sh

    ant run

The program will write the contents of your |S3| object to your command window.

