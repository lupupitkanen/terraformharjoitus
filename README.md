# terraformharjoitus
Harjoitellaan terraform infra-automaatiota Google Cloud Platformin kanssa

# Google Cloud kirjautuminen
```
gcloud auth application-default login
```

# Projektin hakemisto
```
cd
cd projects
mkdir omaprojekti
cd omaprojekti
```

# variables.tf tiedosto
```
variable "project_name" {}
variable "billing_account" {}
variable "region" {
  default = "europe-north1"
}
variable "machine_type" {
  default = "n1-standard-1"
}

variable "cloudflare_email" {}
variable "cloudflare_token" {}
variable "cloudflare_zone" {}
```

# project.tf tiedosto
```
provider "google" {
 region = "${var.region}"
}

resource "random_id" "id" {
 keepers = {
  var.project_name = "${var.project_name}"
 }
 byte_length = 4
 prefix      = "${var.project_name}-"
}

resource "google_project" "project" {
  name            = "${var.project_name}"
  project_id      = "${random_id.id.dec}"
  billing_account = "${var.billing_account}"
}

resource "google_project_services" "project" {
 project = "${google_project.project.project_id}"
 services = [
   "compute.googleapis.com",
   "oslogin.googleapis.com",
   "storage-api.googleapis.com"
 ]
}

output "project_id" {
  value = "${google_project.project.id}"
}
```

# compute.tf tiedosto
```
resource "google_compute_address" "default" {
  name = "${var.project_name}-ip"
  project = "${google_project_services.project.project}"
  region = "${var.region}"
}

resource "google_compute_instance" "default" {

 project = "${google_project_services.project.project}"
 zone = "${var.region}-a"
 allow_stopping_for_update = "true"
 name = "${var.project_name}-web"
 machine_type = "${var.machine_type}"

 boot_disk {
   initialize_params {
     size = "10"
     image = "centos-7"
     type = "pd-ssd"
   }
 }

 network_interface {
   network = "default"
   access_config {
     nat_ip = "${google_compute_address.default.address}"
   }
 }

 metadata {
   "ssh-keys" = "vagrant:${file("~/.ssh/id_rsa.pub")}"
 }

# metadata_startup_script = "${file("startup.sh")}"

 service_account {
   scopes = ["userinfo-email", "compute-ro", "storage-full", "cloud-platform"]
 }

}

output "compute_instance_nat_ip" {
  value = "${google_compute_address.default.address}"
}
```

# cloudflare.tf tiedosto
```
provider "cloudflare" {
  email = "${var.cloudflare_email}"
  token = "${var.cloudflare_token}"
}
```

# firewall.tf tiedosto
```
data "google_compute_network" "default" {
  project = "${google_project_services.project.project}"
  name = "default"
}

resource "google_compute_firewall" "allow_icmp" {
    name = "allow-icmp"
    project = "${google_project_services.project.project}"
    network = "default"

    allow {
        protocol = "icmp"
    }

    source_ranges = ["0.0.0.0/0"]
}

data "cloudflare_ip_ranges" "cloudflare" {}

resource "google_compute_firewall" "allow_cloudflare" {
    name = "allow-cloudflare"
    project = "${google_project_services.project.project}"
    network = "default"

    allow {
        protocol = "tcp"
        ports = ["80"]
    }
    allow {
        protocol = "tcp"
        ports = ["443"]
    }

    source_ranges = ["${data.cloudflare_ip_ranges.cloudflare.ipv4_cidr_blocks}"]
    target_tags = ["web"]
}
```

# Ympäristömuuttujat

- TF_VAR_billing_account: Google Cloud Console -> Billing
- TF_VAR_cloudflare_token: Cloudflare -> My Profile -> API Keys -> Global API Key
- TF_VAR_cloudflare_zone: Oma domain nimesi, esim. sinundomain.online

```
export TF_VAR_billing_account=
export TF_VAR_project_name=
export TF_VAR_region=europe-north1
export TF_VAR_machine_type=n1-standard-1

export TF_VAR_cloudflare_email=
export TF_VAR_cloudflare_token=
export TF_VAR_cloudflare_zone=
```

# Terraform komennot
```
terraform init
terraform plan
terraform apply
```

# Lisäkomentoja
```
# Katso outputit
terraform output

# Kirjaudu palvelimelle
ssh <ip-osoite>

# Vaihda palvelimen tyyppi/koko
export TF_VAR_machine_type=n1-standard-2
terraform plan
terraform apply

# POISTA KAIKKI
terraform destroy

# Luo uudlleen puhtaalta pöydältä

terraform plan
terraform apply
```
