ARG FRAPPE_BRANCH=v13.21.0
FROM node:14-buster
ARG FRAPPE_BRANCH=v13.21.0

COPY install_frappe.sh /install_frappe
COPY install_erpnext.sh /install_erpnext
COPY install_app.sh /install_app
COPY prepare_production.sh /prepare_production

# Install frappe
RUN /install_frappe ${FRAPPE_BRANCH} && \
  # Install erpnext
  /install_erpnext && \
  # Install required apps with following command
  # /install_app <app_name> <repo> <app_branch>
  /install_app gallon https://github.com/lapillaga/gallon main && \
  /install_app jci_website https://github.com/tenpisoft/jci_website.git main && \
  /install_app lapillaga_frappe https://github.com/lapillaga/lapillaga_frappe.git main && \
  # Cleanup for production
  /prepare_production

FROM frappe/erpnext-nginx:${FRAPPE_BRANCH}

COPY --from=0 /home/frappe/frappe-bench/sites/ /var/www/html/
COPY --from=0 /rsync /rsync

# Append list of installed to apps.txt
RUN echo -n "gallon" >> /var/www/html/apps.txt
RUN echo -n "jci_website" >> /var/www/html/apps.txt
RUN echo -n "lapillaga_frappe" >> /var/www/html/apps.txt

VOLUME [ "/assets" ]

ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["nginx", "-g", "daemon off;"]
