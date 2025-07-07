# Debug and Solve Issues during Practical

Here will be some list of issues that you might come across, so beforehand I will be trying to mention and fix them.

## Prevent gcloud from using Putty (on windows)

This issue is already discussed in [StackExchange](https://superuser.com/questions/1281977/prevent-gcloud-from-using-putty-google-cloud-sdk-on-windows), you can check that out or follow the below instructions:

### Steps to SSH into GCP VM instance

You can login to the GCP VM using MobaXterm, Windows Terminal or any other command-prompt application which supports ssh. All you need is the **private key** and the **external IP** of your GCP instance(aka GCP VM).

1. **Locate the Private key**:

    ```bash
    gcloud compute ssh your-instance-name --dry-run
    ```

    - From the output, you'll notice the private key used in the command will end with a `.ppk` extension (`ppk` stands for a putty-private-key).

    - If you browse the **File Explorer** in the *same path*, you'll find another file **without the `.ppk` extension**, that's the private key in the *standard format*.

    Private key is usually `C:\Users\<username>\.ssh\google_compute_engine`

2. **Find out your VM's external IP using 2 ways**:

    - Log in to your VM normally and find out its external IP using the command

      ```bash
      curl ifconfig.co
      ```

    - You can also find both external IP in the [GCP Console page](https://console.cloud.google.com/compute/instances) (possible only if you've admin rights).

3. Now you can SSH into your VM from your favourite shell:

    ```bash
    ssh -i <path to private key without the (.ppk) extension> user@your_instance_IP
    ```

---
