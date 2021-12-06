ARG FRAPPE_BRANCH=v13.15.0
FROM frappe/erpnext-worker:${FRAPPE_BRANCH}

RUN install_app gallon https://github.com/lapillaga/gallon main
RUN install_app jci_website https://github.com/tenpisoft/jci_website.git main
